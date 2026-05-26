# Agent Evals

> **Audience.** Engineers and tech leads responsible for the quality bar of an agentic feature. Anyone whose answer to "is the agent better than last week?" should be a number, not a feeling. **Scope.** The *engineering* of agent-specific evaluation — trajectory eval, step-level eval, outcome-only eval, tool-call accuracy, cost-aware sampling, and the production-trace-to-eval-case pipeline. Composes on top of the broader eval engineering practice (the sibling `eval-engineering/` folder). Not the eval gate at the deployment boundary (see [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md)). Not the LLM-judge depth (see [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Evaluating an agent is substantially harder than evaluating a single-call LLM feature. A single-call eval has a clean shape: input → expected output; the model's output is compared to the expected output via a scoring function. Tens of thousands of golden cases run in minutes; the score is one number; the gate is sharp.

An agent's eval has neither shape:

- The *output* is not just a final answer; it's a trajectory (which tools called, in what order, with what arguments, with what intermediate results, with what cost, with what duration).
- The *expected* trajectory is not unique — many trajectories produce the same final answer; only some are good (cheap, fast, complete).
- The *score* is multi-dimensional: outcome correctness, trajectory quality, cost efficiency, latency, escalation appropriateness.
- The *cases* are expensive to run: each eval case may invoke 5–20 LLM calls; a 1000-case eval is 5,000–20,000 LLM calls.

Teams that don't engineer agent eval ship agents whose quality is measured by anecdote — "this looked better in our spot-checks." The first major regression is the one users surface in production. The eval gate that should have caught it didn't exist.

This document is opinionated about four things:

1. **Eval has three layers; agents need all three.** Outcome eval (did the agent get the right answer?), trajectory eval (was the path good?), and step-level eval (was each individual decision right?). Each layer answers a different question; relying on only one misses real regressions.
2. **Production traces are the most valuable eval cases.** Curated golden cases drift away from production reality. Production traces capture the long tail and the recent issues. The pipeline from trace to eval case is itself a piece of engineering.
3. **Eval costs real money; budget for it.** A 1000-case agent eval run costs $10–$100 (or more for expensive features). Run the eval gates on every promotion; run continuous eval over production samples for drift detection. The budget is recoverable in fewer cost incidents.
4. **The eval is the source of truth for the agent's behaviour.** If a behaviour isn't in the eval, the team doesn't know whether it works. Engineering decisions (prompt change, model swap, tool refactor) are validated against eval; their justification is the eval delta.

Structure: (2) the three eval layers; (3) outcome eval for agents; (4) trajectory eval; (5) step-level eval and tool-call accuracy; (6) cost-aware and budget-aware sampling; (7) the production-trace-to-eval pipeline; (8) eval gates, continuous eval, and drift detection; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The three eval layers

Each layer answers a different question. Together they cover the agent's quality surface.

### 2.1 Outcome eval — "did the agent get the right answer?"

The agent's final response is compared against an expected outcome. Methods:

- **Exact match / structured comparison.** When the agent's output is structured (a number, a category, a JSON shape).
- **Reference comparison.** Comparing free-text output against a reference answer; via embedding similarity, BLEU/ROUGE, or LLM-as-judge.
- **Rubric-based scoring.** An LLM judge scores along rubric dimensions (correctness, completeness, tone, citation accuracy).

What it tells you: whether the agent achieves its purpose.

What it doesn't tell you: whether the agent achieved it efficiently, or for the right reasons, or with the right intermediate decisions.

### 2.2 Trajectory eval — "was the path good?"

The agent's trajectory (sequence of tool calls, decisions, intermediate results) is evaluated. Methods:

- **Trajectory-level rubric.** An LLM judge reads the trajectory and scores along dimensions (was the tool selection reasonable? was the agent efficient? did it avoid unnecessary loops?).
- **Trajectory comparison.** Against a reference trajectory or against historical trajectories on similar inputs.
- **Trajectory pattern detection.** Specific anti-patterns the agent should not exhibit (looping, retrying without strategy, calling redundant tools).

What it tells you: whether the agent's reasoning and approach were good.

What it doesn't tell you: per-step quality details (those need step-level eval).

### 2.3 Step-level eval — "was each individual decision right?"

Individual decisions within the trajectory are evaluated:

- **Tool selection accuracy.** When the agent picked tool X, was X the right choice given the state at that turn?
- **Argument correctness.** Were the arguments well-formed and appropriate?
- **Tool-result interpretation.** Did the agent correctly read the tool's result and proceed appropriately?
- **Termination decision.** Did the agent terminate at the right moment (not too early, not too late)?

What it tells you: where in the trajectory the agent makes mistakes (when it does).

What it doesn't tell you: whether the end result is good (you can have correct steps but a wrong outcome from a missing step, or vice versa).

### 2.4 The three together

| Question | Layer | Method |
| --- | --- | --- |
| "Does the agent work?" | Outcome | Reference comparison |
| "Is the agent inefficient?" | Trajectory | Cost / turn-count / pattern detection |
| "Where does the agent go wrong?" | Step-level | Per-decision rubric |
| "Did the recent change regress?" | All three | Delta from baseline |

A complete eval runs all three. A starting team may begin with outcome eval; adding trajectory and step-level eval as the practice matures.

### 2.5 What the layers cost

Per 1000-case eval, rough orders of magnitude:

- Outcome eval (LLM-as-judge over final answers): $5–$30.
- Trajectory eval (LLM-as-judge over trajectories): $20–$100 (trajectories are long; judge prompts are long).
- Step-level eval (per-turn judge or rule-based checks): $30–$200 (proportional to total turns).

A complete eval of a 1000-case set: $55–$330. Worth budgeting; less than the incident the eval would catch.

---

## 3. Outcome eval for agents

The simplest layer; still warrants discipline.

### 3.1 The golden set

A curated set of (input, expected_outcome) pairs. For an agent:

- **Input.** The user-facing input the agent receives (a question, a task, a request).
- **Expected outcome.** Either an exact answer (when the task has one), a reference answer (when freer-form), or a rubric (when scoring against criteria).
- **Metadata.** Tags for categorising (case type, difficulty, customer persona). Used for slice analysis.

Cases are added when:

- A new feature lands.
- A bug is fixed (the failing input is added).
- A new edge case is encountered in production (via the trace-to-eval pipeline).

Cases are removed when:

- The functionality they cover is removed.
- They become irrelevant due to product changes.
- They are subsumed by more comprehensive cases.

The golden set grows over time. Healthy size for a moderate-complexity agent: 500–5000 cases.

### 3.2 Outcome scoring

Per case, the agent's outcome is scored. Methods (per [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md)):

- **Structured output.** Exact match or schema-based comparison; deterministic.
- **Free-text reference.** LLM judge with a clear rubric ("does the output answer the question correctly, completely, and concisely?").
- **Rubric-based.** Multi-dimensional rubric ("correctness 0-5, completeness 0-5, tone 0-5"); aggregate via weighted sum or per-dimension reporting.

The score per case feeds the aggregate report.

### 3.3 The aggregate

- **Overall score.** Mean across cases.
- **By slice.** Score broken down by tag (case type, difficulty, persona).
- **Failure list.** Cases where the score is below threshold; engineered into actionable list.

The aggregate is the headline; the failure list is the working surface for improvement.

### 3.4 The gate

The eval gate (per [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md)) blocks promotion when:

- Overall score drops > X% vs baseline.
- Any critical-slice score drops > Y% (more sensitive than overall).
- New failures appear in cases that previously passed (regression detection).

The gate's thresholds are calibrated to balance false-positive (blocking good changes) vs false-negative (allowing regressions). Quarterly review.

### 3.5 The outcome eval's limits

Outcome eval misses:

- **Cost regressions.** The agent produces the right answer but uses 3× the cost.
- **Latency regressions.** The agent produces the right answer but takes twice as long.
- **Trajectory deterioration.** The agent produces the right answer but via a worse path (more loops, more retries, more tool calls).
- **Step-level regressions in cases that still pass overall.** The agent's tool selection accuracy dropped 5% but it still gets to the right answer via repair loops.

For these, trajectory and step-level eval are needed.

---

## 4. Trajectory eval

The agent's path matters, not just the destination.

### 4.1 What's evaluated

For each case:

- **Turn count.** How many turns did the agent take?
- **Cost.** Total cost of the invocation.
- **Latency.** Wall-clock time.
- **Tool-call count and distribution.** Which tools called, how many times.
- **Repair-loop count.** How often did the agent attempt to repair a malformed response or tool error?
- **Escalation flag.** Did the agent escalate? Was it the right call?
- **Trajectory rubric score.** LLM judge scores the trajectory along rubric dimensions.

### 4.2 The trajectory rubric

An LLM judge reads the trajectory and scores:

- **Efficiency.** Was the agent direct, or did it meander?
- **Tool selection appropriateness.** Were tools chosen well at each turn?
- **Recovery quality.** When errors occurred, did the agent recover well?
- **Termination appropriateness.** Did the agent terminate at the right point?

```
Judge prompt template:

You are evaluating an AI agent's trajectory on the following task:
[task description]

The agent's trajectory was:
[summarised trajectory]

Score the trajectory on these dimensions, each 0-5:
1. Efficiency: did the agent take a direct path?
2. Tool selection: were tools chosen appropriately at each turn?
3. Recovery: when errors occurred, did the agent handle them well?
4. Termination: did the agent terminate at the right moment?

Return JSON: {efficiency: N, tool_selection: N, recovery: N, termination: N, notes: "..."}
```

The judge runs per case; results aggregate to per-dimension averages.

### 4.3 Trajectory anti-pattern detection

Beyond rubric, specific anti-patterns are detected programmatically:

- **Loop.** Same tool with same arguments called > N times.
- **Thrash.** Quick succession of alternating tools without progress.
- **Stuck.** Many turns with no state change.
- **Unnecessary repair.** Repair attempts that produced the same outcome as the original call.
- **Skipped tool.** A tool the case's reference trajectory used was not called.

Per-anti-pattern rates feed dashboards and alerts.

### 4.4 Trajectory comparison

For cases where a reference trajectory exists:

- Trajectory similarity (sequence alignment, embedding similarity).
- Cost ratio (current vs reference).
- Turn-count ratio.

Used for regression detection — if the new trajectory is significantly different from the reference on cases that previously passed cleanly, investigation is warranted.

### 4.5 The trajectory rubric judge's quality

The judge is itself an LLM; it has its own quality. Discipline:

- **Calibrated against humans.** A sample of judge scores is re-scored by humans; the judge's agreement rate is the calibration metric.
- **Stable judge prompt.** The rubric prompt is versioned (per [prompt-versioning.md](../prompt-engineering/prompt-versioning.md)); changes are eval-tested.
- **Drift detection.** Judge scores on a stable subset are monitored over time; drift indicates the judge needs recalibration.

The judge's quality directly affects the trajectory eval's value. Invest in it.

---

## 5. Step-level eval and tool-call accuracy

The most detailed layer; the most expensive; the most diagnostic.

### 5.1 Per-turn evaluation

Each turn in the trajectory is a decision point. The eval examines each:

- **The state at the turn.** What did the agent know?
- **The decision.** What did the agent decide to do?
- **The reference decision.** What should the agent have decided?

The reference decision is hard to specify in general. Approaches:

- **Curated cases with reference trajectories.** Specific cases for which the eval team has decided the right per-turn decisions.
- **Heuristic checks.** "At a turn where the agent has a not-found error from a search, the next decision should not be the same search with the same arguments."
- **LLM judge per turn.** Slow and expensive but flexible. Useful for ambiguous cases.

### 5.2 Tool-call accuracy

The most operationally important step-level metric. Per [tool-architecture.md](./tool-architecture.md) section 9.6.

**Tool selection accuracy.** Across all turns where a tool was called, what fraction picked the "right" tool?

The "right" tool is defined per case. Approaches:

- **Curated cases with reference tool per turn.** Specific golden cases tagged with the expected tool at each turn.
- **Heuristic rules.** "If the agent has just received a patient_id, the next tool with that patient_id as argument should be from the patient-data tool group."
- **Negative cases.** "On this input, the agent should NOT call X."

Tool selection accuracy is reported per tool, per case slice, and overall. Quarterly target: > 95% overall, > 90% per-tool.

**Argument accuracy.** When the tool was called, were the arguments well-formed and contextually appropriate?

- Format checks (matches schema).
- Value plausibility checks.
- Cross-call consistency (e.g., the patient_id from a prior call is reused correctly).

### 5.3 Step-level regression detection

A common failure mode: a prompt change improves outcome eval marginally but causes the agent to make different tool choices that look fine in isolation but compound over many cases. Step-level eval catches this:

- Tool selection accuracy on the eval set has dropped from 96% to 91%.
- Outcome eval is up 2%.
- The interpretation: the agent is getting to the right answer via worse paths; future regression likely.

Without step-level eval, this is invisible.

### 5.4 The "tool with the agent" eval pattern

A pattern from [tool-architecture.md](./tool-architecture.md): for each tool, eval includes:

- Cases where the tool *should* be called → verify it is.
- Cases where the tool *should not* be called → verify it isn't.
- Cases where the tool's description should disambiguate it from a similar tool → verify the right one is chosen.

The tool-with-agent eval is the bridge between tool architecture and agent eval.

### 5.5 Step-level eval's cost

The most expensive layer. Per 1000-case eval, ~$100–$500. Discipline:

- **Sample.** Step-level eval on a representative sample (10–20% of cases) rather than every case.
- **Trigger.** Full step-level eval only on promotion (eval gate); continuous eval uses outcome + trajectory only.
- **Slice focus.** Step-level eval on slices known to be sensitive (high-stakes cases, novel features, recently changed flows).

---

## 6. Cost-aware and budget-aware sampling

Eval has cost; budget for it.

### 6.1 The cost equation

Per eval run:

```
eval_cost = sum over cases of (
    agent_invocation_cost  // running the agent on the case
  + outcome_judge_cost     // LLM judge for outcome
  + trajectory_judge_cost  // LLM judge for trajectory
  + step_judge_cost        // LLM judge for steps (if applicable)
)
```

A 1000-case eval with all layers can be $100–$500. Multiple runs per day during heavy development.

### 6.2 Sampling strategies

To bound cost while preserving signal:

**Tiered eval set.**
- Tier 1: critical cases (200–500). Run on every commit / promotion. Full layers.
- Tier 2: broad cases (1000–3000). Run on promotion. Outcome + trajectory.
- Tier 3: extensive cases (5000–20000). Run nightly. Outcome only.

**Slice-stratified sampling.** Sample from each slice (case type, difficulty, persona); ensure coverage of important slices.

**Recency-weighted sampling.** Recent additions (from production traces) get higher sampling rate to validate the addition's quality.

**Adaptive sampling.** Spend more eval budget on cases where the model recently regressed.

### 6.3 The eval gate vs continuous eval

Two distinct uses of eval, with different cost profiles:

| Use | Cadence | Layers | Eval size | Cost / run |
| --- | --- | --- | --- | --- |
| Eval gate (promotion) | Per promotion | All | Tier 1 + Tier 2 | $20–$200 |
| Continuous eval (production) | Hourly / daily | Outcome + light trajectory | Sample of recent production traces | $5–$50 / day |
| Comprehensive eval (release) | Weekly / monthly | All | Tier 1 + Tier 2 + Tier 3 | $200–$1500 |

The gate is the most important; continuous catches drift; comprehensive verifies the long tail.

### 6.4 The eval budget

Per feature, allocate an eval budget. Reasonable benchmark: 5–15% of the feature's LLM cost.

For Meridian's care-coordinator at ~$15k/month LLM cost: eval budget ~$1500/month. Covers all three uses comfortably.

### 6.5 The "we can't afford eval" objection

Some teams cite eval cost as a reason to skip it. The framing is wrong: the alternative cost is incident cost. Incidents prevented by eval pay for the eval many times over. The framing for budget conversations: "the eval line item is part of the cost of running this feature; it's smaller than the average incident's cost."

---

## 7. The production-trace-to-eval pipeline

Production traces are the most valuable eval material; the pipeline that promotes them to eval cases is itself important engineering.

### 7.1 Why production traces

- They reflect real user input distribution (curated cases drift).
- They expose long-tail patterns the team didn't anticipate.
- They include the most recent issues (production is always ahead of the eval team).
- They're free to capture (the trace is generated regardless).

### 7.2 The promotion pipeline

```
Production trace → reviewer (human or LLM judge with human verification) → triaged →
    candidate eval case (input + observed outcome + tag) →
    review (was this a good case to add? what's the expected outcome?) →
    eval case (added to the golden set)
```

Each step has discipline:

- **Reviewer.** What signal triggers promotion? Failed outcome, low judge score, escalation, budget breach, customer complaint.
- **Triage.** Categorise by type; tag for slice analysis.
- **Expected outcome specification.** The reviewer (often a domain expert) specifies what the right outcome should have been.
- **Review.** Quality bar before adding to the golden set.

### 7.3 What gets promoted

Not every trace is a useful eval case. Discipline on selection:

- **Failures and near-failures.** Outcome was wrong, judge scored low, agent escalated, agent breached budget.
- **Novel input patterns.** Inputs that don't match existing slices.
- **High-stakes successes.** Cases where the agent did well on something important; pin the behaviour with an eval case.
- **Customer complaints.** Direct user feedback (when available) → eval case.

What doesn't get promoted:

- **Duplicates.** Cases similar to existing eval cases add little.
- **Edge cases that won't recur.** Some inputs are genuinely one-offs.
- **Ambiguous cases.** If the team can't agree on what the right outcome is, the case is too noisy.

### 7.4 Privacy and PII handling

Production traces contain PII / PHI. Discipline:

- **Redaction at promotion.** PII fields are redacted or substituted with synthetic values before the case enters the eval set.
- **Access controls.** The eval set's storage has appropriate access controls.
- **Audit.** Promotion actions are logged.

Without the privacy layer, promotion is blocked by legal. With it, promotion scales.

### 7.5 The vendor integration

LangSmith, Braintrust, and similar tools provide trace-to-eval-case features. Use them where they fit. Build a thin wrapper if the vendor doesn't support the exact pattern.

### 7.6 The "trace promotion" cadence

Weekly trace-promotion sessions in a mature operation:

- Review failures / low-quality / escalation traces from the past week.
- Decide which to promote.
- Specify expected outcomes; tag.
- Add to the eval set.
- Run the eval gate against the new cases on the next promotion.

A typical week might promote 5–30 cases.

---

## 8. Eval gates, continuous eval, and drift detection

How the eval surface integrates with the broader CI/CD and operational practice.

### 8.1 The eval gate

Per [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md). Summary for agents:

- Runs on every promotion candidate (prompt change, model change, tool change).
- Tier 1 + Tier 2 evaluated.
- Pass/fail on:
  - Outcome score within tolerance.
  - Critical slice scores within tolerance.
  - No new regressions.
  - Trajectory metrics (cost, turn count) within tolerance.

A failed gate blocks promotion; investigation is required.

### 8.2 Cost regression as gate condition

Per [agent-cost-control.md](./agent-cost-control.md), the gate's tolerances include cost. A prompt change that improves outcome 1% but raises cost 50% is blocked; the change is reworked.

### 8.3 Continuous eval over production

A sample of production traces is continuously eval'd:

- LLM-as-judge scores outcome quality.
- Trajectory metrics aggregate.
- Quality trends are reported.

Used for:

- **Drift detection.** Quality drops below threshold trigger investigation.
- **Pre-incident signal.** Often drift precedes a user-visible incident; catching it early prevents it.
- **Model upgrade signal.** When the provider releases a new model, continuous eval on the canary helps decide whether to upgrade.

### 8.4 Drift detection thresholds

- Day-over-day quality change > 3%: investigation.
- Week-over-week quality change > 5%: alert.
- Significant slice-specific drop: immediate alert.

The thresholds are calibrated against historical noise; should be tighter for high-stakes features.

### 8.5 Shadow eval

For high-risk changes, shadow eval: the candidate runs in parallel with the production version on the same inputs; outcomes are compared. The shadow doesn't affect users; it's pure eval signal.

Used for:

- Model upgrades.
- Major prompt rewrites.
- Tool catalog changes.

The shadow's results inform whether to promote.

### 8.6 Eval reproducibility

Eval runs must be reproducible:

- Versioned eval set (the cases at a point in time).
- Versioned agent (the prompts, tools, model pin at a point in time).
- Versioned judge (the rubric prompt).
- Recorded seed for stochastic elements.

Without reproducibility, eval comparisons are meaningless ("did the score actually change, or was it noise?").

---

## 9. Worked Meridian example

Meridian's care-coordinator agent eval.

### 9.1 The eval set

- **Total cases:** ~2400.
- **Tier 1 (critical):** 380 cases. Run on every promotion. All three layers.
- **Tier 2 (broad):** 1850 cases. Run on promotion. Outcome + trajectory.
- **Tier 3 (extensive):** added as production-trace promotions over time. Outcome only.

Tags include: case type (clinical question / scheduling / lookup / escalation / multi-turn), difficulty (easy / medium / hard / edge), persona (primary-care / specialist / care-manager).

### 9.2 Eval cost

- Per gate run: ~$60 (Tier 1 + Tier 2 with outcome + trajectory; Tier 1 with step-level).
- Continuous eval daily: ~$20 (sample of recent production).
- Comprehensive monthly: ~$400 (all tiers, all layers).
- Total monthly eval cost: ~$1400 (the feature's LLM cost is ~$15k/month; eval is ~9%).

### 9.3 Outcome eval

LLM-judge (claude-sonnet-4-6) scores along rubric dimensions: correctness, completeness, clinical-appropriateness, tone. Per-dimension scores 0–5; weighted aggregate is the case score.

- Current overall mean: 4.31.
- Baseline (3 months ago): 4.28.
- Significant slices: clinical questions 4.40, scheduling 4.55, lookup 4.28, escalation 4.05, multi-turn 4.18.

### 9.4 Trajectory eval

Per case:

- Turn count distribution: median 3, P95 7, P99 12.
- Cost distribution: per [agent-cost-control.md](./agent-cost-control.md) section 9.1.
- Anti-pattern detection: loop rate 0.2%, thrash rate 0.1%, unnecessary repair rate 0.3%.
- Trajectory rubric mean: 4.45 (judge scores efficiency, tool selection, recovery, termination).

### 9.5 Step-level eval

Run on Tier 1 (380 cases) on every promotion:

- Tool selection accuracy: 96.4% overall; 92.1% on multi-turn slice (the hardest).
- Argument accuracy: 98.2% overall.
- Per-tool: fetch_patient 99.1%; search_clinical_notes 94.2%; propose_followup 95.5%; escalate 91.5% (lower; escalation decisions are subjective).

### 9.6 The eval gate

Promotion gate:

- Outcome aggregate within 2% of baseline. Tolerance is tight (production quality matters).
- No critical-slice drops > 5%.
- No new regressions (cases that passed baseline and fail now).
- Trajectory metrics within tolerance (cost +20%, turn count +30%).
- Step-level accuracy within 2 percentage points.

Promotions blocked: ~8 per year for tolerance breaches; ~3 per year for regressions. All blocks investigated; ~60% lead to rework, ~40% to tolerance adjustment after discussion.

### 9.7 Production-trace pipeline

Weekly trace review:

- Failed-outcome traces (LLM-judge score < 3).
- Escalation traces (was the escalation right?).
- Budget-breach traces (could it have been avoided?).
- Customer-complaint-linked traces (from the support team's tickets).

Average promotion: ~12 cases / week. The eval set grew from 1100 cases at launch to 2400 over 14 months.

### 9.8 Continuous eval

Daily LLM-judge run over a 200-trace sample of the day's production:

- Tracked daily quality score over time.
- Slice breakdown.
- Drift alert if day-over-day > 3% or week-over-week > 5%.

One drift event in the last 6 months: a 4% drop over 3 days that traced to a tool's upstream change subtly affecting retrieval quality. Investigation took 2 days; fix landed in week 2; quality recovered.

### 9.9 What worked

- **Tiered eval set.** The Tier 1 / 2 / 3 separation made the eval gate fast and the comprehensive monthly possible.
- **Production-trace promotion.** Kept the eval set current with production reality.
- **Step-level eval on Tier 1.** Caught tool-selection regressions that outcome eval missed.
- **Continuous eval.** Caught the upstream-tool-change incident before it became user-visible.

### 9.10 What didn't work initially

- **Single-layer eval (outcome only).** Missed several trajectory regressions in the first year; added trajectory eval after.
- **All-cases-every-promotion.** Was too slow; moved to tiered approach.
- **Judge prompt drift.** The first judge prompt drifted over a few months; tightened versioning discipline.

### 9.11 The PIR-to-eval loop

Each post-incident review produces:

- New eval cases (the incident's pattern, with expected behaviour).
- Updated eval gate (new threshold or new check if the incident exposed a gap).
- Sometimes a new alert (if observability missed the incident).

The loop is the discipline that prevents recurrence.

---

## 10. Anti-patterns

### 10.1 "Outcome eval only"

Only outcome is measured. Trajectory and step-level regressions invisible. The agent's cost and turn-count drift unnoticed.

**Corrective.** All three layers per section 2.

### 10.2 "Eval is curated cases that don't reflect production"

The golden set was built once at launch; production drifts; the eval no longer represents real usage.

**Corrective.** Production-trace promotion per section 7.

### 10.3 "Eval gate has no cost or trajectory thresholds"

The gate passes a change that improves outcome 1% but raises cost 50%. The cost regression reaches production.

**Corrective.** Multi-dimensional gate per section 8.1.

### 10.4 "No continuous eval"

Production quality is measured only at promotion. Drift is invisible until it becomes a user-visible incident.

**Corrective.** Continuous eval per section 8.3.

### 10.5 "Judge prompt drift"

The LLM judge's prompt evolved over time without versioning; old eval scores aren't comparable to new.

**Corrective.** Versioned judge prompt per section 4.5.

### 10.6 "Eval isn't reproducible"

Re-running an eval produces different scores; can't tell signal from noise.

**Corrective.** Reproducibility discipline per section 8.6.

### 10.7 "Production traces never promoted"

The promotion pipeline doesn't exist; the eval set ossifies.

**Corrective.** Trace-promotion cadence per section 7.6.

### 10.8 "Eval cost is too high so we skip it"

The team treats eval as optional; ships changes without gate.

**Corrective.** Eval budget per section 6.5; recoverable through prevented incidents.

---

## 11. Findings (sprint-assignable)

### AGT-EVAL-001 — Severity: Critical
**Finding.** Agent has no eval; quality changes are detected by user complaints.
**Recommendation.** Outcome eval per section 3; golden set of 200+ cases; eval gate.
**Owner.** ai-platform-eng + feature-team, sprint N+1.

### AGT-EVAL-002 — Severity: Critical
**Finding.** Eval gate has no cost or trajectory thresholds; cost regressions reach production.
**Recommendation.** Multi-dimensional gate per section 8.1; cost-delta and trajectory checks.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-EVAL-003 — Severity: High
**Finding.** Outcome eval only; trajectory and step-level regressions invisible.
**Recommendation.** Add trajectory eval per section 4; add step-level eval on Tier 1 per section 5.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-EVAL-004 — Severity: High
**Finding.** Eval set is static; production drift unrepresented.
**Recommendation.** Production-trace pipeline per section 7; weekly promotion cadence.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-EVAL-005 — Severity: High
**Finding.** No continuous eval; production quality drift invisible.
**Recommendation.** Continuous eval per section 8.3; drift alerts per section 8.4.
**Owner.** ai-platform-eng + ops, sprint N+2.

### AGT-EVAL-006 — Severity: High
**Finding.** Tool selection accuracy not measured; tool surface quality unknown.
**Recommendation.** Tool selection accuracy per section 5.2; reported per tool, per slice, monthly.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-EVAL-007 — Severity: Medium
**Finding.** Eval set is untiered; full eval on every promotion is too slow or too expensive.
**Recommendation.** Tiered eval set per section 6.2; Tier 1 for gate, Tier 2 + 3 for less-frequent.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-EVAL-008 — Severity: Medium
**Finding.** Judge prompt drifts; eval scores not comparable across time.
**Recommendation.** Versioned judge prompt per section 4.5; calibration against human scores periodically.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-EVAL-009 — Severity: Medium
**Finding.** Eval not reproducible; can't distinguish signal from noise.
**Recommendation.** Reproducibility discipline per section 8.6; pinned eval set, agent version, judge version.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-EVAL-010 — Severity: Medium
**Finding.** PII / PHI in production traces blocks promotion; eval set stale.
**Recommendation.** Redaction layer per section 7.4; promotion pipeline includes redaction step.
**Owner.** ai-platform-eng + privacy, sprint N+3.

### AGT-EVAL-011 — Severity: Medium
**Finding.** Anti-pattern detection (loop, thrash, unnecessary repair) not in eval; trajectory pathologies missed.
**Recommendation.** Programmatic checks per section 4.3.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-EVAL-012 — Severity: Medium
**Finding.** Eval costs not tracked; eval budget unclear; conversations about eval scope are noisy.
**Recommendation.** Per-feature eval budget per section 6.4; cost tracked; quarterly review.
**Owner.** ai-platform-eng + finance, sprint N+3.

### AGT-EVAL-013 — Severity: Medium
**Finding.** No shadow eval for high-risk changes; model upgrades and major rewrites land without parity validation.
**Recommendation.** Shadow eval per section 8.5 for high-risk changes.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-EVAL-014 — Severity: Medium
**Finding.** Slice-level reporting absent; aggregate hides slice-specific regressions.
**Recommendation.** Slice tags per section 3.1; per-slice score in gate.
**Owner.** ai-platform-eng + feature-team, sprint N+4.

### AGT-EVAL-015 — Severity: Low
**Finding.** PIR's don't drive eval set updates; eval doesn't learn from incidents.
**Recommendation.** PIR-to-eval loop per section 9.11.
**Owner.** ai-platform-eng + ops, sprint N+4.

### AGT-EVAL-016 — Severity: Low
**Finding.** Trajectory rubric judge not calibrated against humans; judge quality unknown.
**Recommendation.** Periodic calibration per section 4.5; agreement rate metric.
**Owner.** ai-platform-eng + domain experts, sprint N+5.

### AGT-EVAL-017 — Severity: Low
**Finding.** No comprehensive monthly eval; long-tail quality unmeasured.
**Recommendation.** Monthly comprehensive run per section 6.3.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-EVAL-018 — Severity: Low
**Finding.** Eval-pipeline failures not alerted; eval silently broken for days.
**Recommendation.** Eval-pipeline health monitoring; alert on missing runs or unusual results.
**Owner.** ai-platform-eng + ops, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team launching a new agent:

- [ ] **Sprint 0 — golden set v1.** 100–300 curated cases covering the main flows.
- [ ] **Sprint 1 — outcome eval.** LLM-judge or structured comparison; aggregate report.
- [ ] **Sprint 1 — eval gate.** Pass/fail on outcome thresholds; blocks promotion.
- [ ] **Sprint 2 — trajectory eval.** Turn count, cost, anti-pattern detection.
- [ ] **Sprint 2 — judge versioning.** Versioned rubric; calibrated.
- [ ] **Sprint 3 — production-trace pipeline.** Reviewer signal; promotion process.
- [ ] **Sprint 3 — continuous eval.** Daily LLM-judge on production sample.
- [ ] **Sprint 4 — step-level eval on Tier 1.** Tool selection accuracy.
- [ ] **Sprint 4 — tiered eval set.** Tier 1, 2, 3.
- [ ] **Sprint 4 — drift alerts.** Day-over-day and week-over-week thresholds.
- [ ] **Ongoing — weekly promotion.** Trace-to-eval cadence.
- [ ] **Ongoing — quarterly tolerance review.** Gate thresholds, judge calibration.

For a team retrofitting eval on an existing agent:

- [ ] **Sprint 0 — eval discovery.** What eval exists? What's the coverage? What's missing?
- [ ] **Sprint 1 — outcome eval if absent.** Build the baseline.
- [ ] **Sprint 1 — gate integration.** Block promotions on outcome regression.
- [ ] **Sprint 2 — trajectory eval.** Catch cost and turn-count regressions.
- [ ] **Sprint 3 — production-trace pipeline.** Get the eval set current.
- [ ] **Sprint 4 — continuous eval.** Drift detection in production.
- [ ] **Sprint 5 — step-level eval.** The most detailed layer.

A team that completes the sequence has quality measured continuously, regressions caught at gate, drift detected before incidents. A team that doesn't ships changes blind and finds out from users.

---

## 13. References

- [agent-engineering-playbook.md](./agent-engineering-playbook.md) — broader practice; section 10 (evaluation).
- [agent-loop-design.md](./agent-loop-design.md) — runner whose trajectory is the eval subject.
- [tool-architecture.md](./tool-architecture.md) — tool selection accuracy; tool-with-agent eval pattern.
- [memory-engineering.md](./memory-engineering.md) — memory eval surface (per memory section 8.5).
- [error-and-partial-failure.md](./error-and-partial-failure.md) — failure-handling eval coverage.
- [agent-cost-control.md](./agent-cost-control.md) — cost as gate dimension.
- [agent-observability.md](./agent-observability.md) — trace-to-eval-case pipeline.
- [multi-agent-coordination.md](./multi-agent-coordination.md) — per-agent + end-to-end eval for multi-agent.
- [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — broader eval practice.
- [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — golden set construction.
- [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md) — LLM-judge depth.
- [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — regression-suite patterns.
- [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — continuous eval in production.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — promotion gate.
- [eval-engineering/eval-of-rag.md](../eval-engineering/eval-of-rag.md) — RAG-specific eval (when agent uses RAG tools).
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — span shape that supplies eval-promotion data.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — drift-detection patterns.
- [prompt-engineering/prompt-versioning.md](../prompt-engineering/prompt-versioning.md) — versioning judge prompts (and agent prompts).
- LangSmith, Braintrust — vendor tools with strong eval / trace integration.
