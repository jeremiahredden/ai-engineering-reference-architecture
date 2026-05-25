# LLM-as-Judge Patterns

> **Audience.** Engineers building or refactoring the judge that scores eval outputs. Tech leads who have heard "the judge said it passed but the answer was clearly wrong" and want a structured discipline. **Scope.** The *engineering* practice of LLM-as-judge for AI eval — judge architectures, calibration, drift detection, specialized judges. Pair with [golden-set-design.md](./golden-set-design.md) (what the judge scores against), [eval-engineering-playbook.md](./eval-engineering-playbook.md) (the broader eval practice), [online-eval-and-feedback.md](./online-eval-and-feedback.md) (the judge in production sampling). **Worked client.** Meridian Health.

---

## 1. Why this document exists

LLM-as-judge is the workhorse pattern for AI eval automation. A separate (typically more capable) model scores each produced answer against a rubric. Done well, the judge enables eval suites that run on every PR, gate releases, sample production traffic, and produce trend data the team trusts. Done poorly, the judge is a noise generator — the team eventually stops trusting the eval results and the entire practice degrades.

The discipline distinguishes "we have a judge running" from "we have a judge running that the team can trust." The latter requires deliberate engineering: judge architecture choice, judge prompt design, calibration against human review, drift detection, recalibration cadence. Without this engineering, the judge produces verdicts the team cannot use to make decisions.

The [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 4 introduces the pattern. This document is the depth — the judge architecture variants, the calibration discipline, the failure modes, and the engineering work that keeps the judge trustworthy over time.

This document is opinionated about three things:

1. **The judge is more capable than the production system.** Same-tier judges agree with the production system's mistakes. Cross-tier (judge more capable than production) is the foundation.
2. **Calibration is non-negotiable.** Without calibration against human review, the judge's verdicts have no validated correlation with reality. Uncalibrated judges produce confident verdicts that may be confidently wrong.
3. **Judge drift is monitored explicitly.** The judge can drift (its prompt becomes stale, its model version shifts, the workload evolves out from under the rubric). Quarterly recalibration is the discipline that keeps drift visible.

Structure: (2) the judge architectures; (3) judge model selection; (4) judge prompt design; (5) calibration against human review; (6) drift detection; (7) specialized judges; (8) integration patterns; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The judge architectures

Three architectures for LLM-as-judge, each with different properties.

### 2.1 Reference-based judge

The judge sees: the question, the expected answer (from the golden set), the produced answer. The judge decides: does the produced answer satisfy the rubric, using the expected answer as the reference standard?

**When right.** When the workload has clearly-defined expected answers and the rubric is specific.

**Trade-off.** Easier to calibrate (the expected answer is the reference); does not work for cases without expected answers (online judging on novel queries).

### 2.2 Reference-free / rubric-based judge

The judge sees: the question, the rubric, the produced answer. The judge decides: does the produced answer satisfy the rubric, scored against the rubric's criteria?

**When right.** When the workload has scoring criteria that can be applied without a reference answer (e.g., "is the answer factually correct"; "does the citation exist"; "does the answer refuse appropriately").

**Trade-off.** Works for online judging on production traffic (no expected answer needed); harder to calibrate (no reference standard).

### 2.3 Pairwise comparison judge

The judge sees: the question, two produced answers (e.g., produced by two different system configurations). The judge decides: which is better?

**When right.** When evaluating relative quality between system configurations (A/B testing, model comparison, prompt comparison).

**Trade-off.** Produces a relative-quality signal but not an absolute pass/fail; useful for choosing between alternatives but not for gating.

### 2.4 The composition pattern

Most production eval practices use multiple judges:

- Reference-based for the offline golden set.
- Reference-free for online judging on production traffic.
- Pairwise for system comparisons during evaluation projects.

Each is calibrated independently. The composition is the engineering work of running them and aggregating results appropriately.

---

## 3. Judge model selection

The judge's model choice is the single most consequential design decision.

### 3.1 The cross-tier requirement

The judge should be *more capable* than the system under test:

- If production runs Sonnet → judge runs Opus.
- If production runs Opus → judge runs Opus (different prompt, different context) or a specialized eval-tuned model.
- If production runs a fine-tuned smaller model → judge runs the frontier model.

Same-tier judges produce silent under-detection. The judge agrees with the system's mistakes because it would make the same mistakes itself.

### 3.2 The cross-provider option

Some teams use a different provider's model as the judge to reduce shared-bias risk:

- Production on Anthropic Claude → judge on OpenAI GPT.
- Production on OpenAI → judge on Anthropic.

The cross-provider judge catches errors a same-provider judge would not detect. The trade-off: maintaining the cross-provider relationship (BAA, contracts, capacity) for a low-volume use case (the judge runs on eval traffic).

### 3.3 The judge model version pinning

The judge model is pinned per [model-registry.md](../model-lifecycle/model-registry.md) and [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md). Aliases are not allowed.

A judge model version change is a recalibration event:
- The new judge model may disagree with the prior judge on cases that were previously calibrated.
- Re-run calibration against human review with the new judge.
- Update the judge prompt if drift is meaningful.

The discipline: judge model changes are PR-reviewed; calibration is verified before the judge is in production use.

### 3.4 The cost consideration

Judge calls are paid like any other model call. For a 200-case eval suite with multiple judge dimensions per case, the cost can be meaningful ($5-15 per full suite run). Considerations:

- Run full judge nightly + on release candidates; subset for per-PR (per [eval-gate-architecture.md](./eval-gate-architecture.md)).
- For online judging, sample (5-10% of production traffic) rather than 100%.
- Cost is justified by the eval discipline's value — typically much less than the cost of a quality regression that would have been caught.

---

## 4. Judge prompt design

The judge prompt is itself an artifact under the team's prompt-engineering discipline.

### 4.1 The prompt structure

A judge prompt has these components:

1. **Role and task.** What the judge is evaluating; what the verdict format is.
2. **The rubric.** The specific criteria the judge is applying.
3. **The case context.** Question, expected answer (if reference-based), produced answer.
4. **Output format.** Structured (JSON) so verdicts can be aggregated programmatically.
5. **Examples (optional).** Few-shot examples of correctly-judged cases (especially helpful for difficult rubrics).

### 4.2 Example judge prompt

```
You are evaluating a produced answer to a clinical question. Your evaluation
will gate clinical-decision-support system releases, so accuracy is critical.

QUESTION:
{question}

EXPECTED ANSWER (semantic reference; the produced answer may vary in wording
but must satisfy all listed criteria):
{expected_answer}

REQUIRED CLAIMS (the produced answer must include each of these claims with
appropriate citation):
{required_claims}

PRODUCED ANSWER:
{produced_answer}

RUBRIC:
1. Factually correct: every claim in the produced answer is supportable by the
   expected answer's content or by other authoritative clinical sources.
2. Complete: the produced answer addresses the question; required claims are
   present.
3. Cites correctly: claims that require citation are cited; citations point to
   real sources; the cited content supports the claim.
4. Appropriate: does not over-claim; does not hallucinate confidence;
   refuses when refusal is warranted.

Evaluate each criterion separately. Return JSON in this exact format:
{
  "factually_correct": {"verdict": "pass" | "fail", "justification": "..."},
  "complete":          {"verdict": "pass" | "fail", "justification": "..."},
  "cites_correctly":   {"verdict": "pass" | "fail", "justification": "..."},
  "appropriate":       {"verdict": "pass" | "fail", "justification": "..."},
  "overall":           "pass" | "fail"
}

The overall verdict is PASS only if all four criteria pass. Otherwise FAIL.
```

### 4.3 The structured-output requirement

The judge's verdict is JSON. The verdict is parsed; aggregations require structured data. Prose verdicts cannot be aggregated.

The judge prompt enforces structure. The judge's output is validated post-hoc; malformed verdicts are flagged.

### 4.4 The justification field

Every verdict has a justification — a one-sentence explanation. The justification is:

- Useful for calibration (when a human reviewer disagrees with the judge, the justification shows the judge's reasoning).
- Useful for debugging (when a case fails unexpectedly, the justification identifies the reason).
- Not used for aggregation (only verdicts are aggregated).

### 4.5 The judge prompt versioning

Judge prompts are versioned per [prompt-versioning.md](../prompt-engineering/prompt-versioning.md). Changes to the judge prompt are:

- Eval-validated (against the calibration set).
- PR-reviewed.
- Versioned and pinned in release manifests (the eval suite has its own release manifest including the judge prompt version).

A judge prompt change is a calibration event — the new prompt may produce different verdicts than the old; the team verifies via human review.

### 4.6 The judge prompt for RAG

For RAG cases (per [eval-of-rag.md](./eval-of-rag.md)), the judge prompt includes the retrieval results and asks separately about citation accuracy and faithfulness:

```
PRODUCED ANSWER:
{produced_answer}

RETRIEVED CHUNKS (the produced answer must be grounded in these):
{retrieved_chunks}

CITED SOURCES (each cited claim must be supported by content in the cited source):
{citations}

Evaluate:
- citation_accuracy: each cited claim is supported by content in its cited source.
- faithfulness: every claim in the produced answer is grounded in some retrieved chunk.
- ...
```

The RAG-specific judge has more context to consume; the cost per case is higher; the rubric is more nuanced.

### 4.7 The judge prompt for agents

For agent cases (per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 11), the judge prompt sees the full agent trajectory (per-turn decisions, tool calls, intermediate results) and scores trajectory quality alongside outcome:

```
PRODUCED TRAJECTORY:
{trajectory_with_per_turn_decisions_and_tool_calls}

FINAL ANSWER:
{produced_answer}

Evaluate:
- outcome: was the final answer correct?
- trajectory: were intermediate decisions reasonable?
- efficiency: did the agent complete in reasonable turn count / cost?
- ...
```

---

## 5. Calibration against human review

Calibration is the foundation. Without it, the judge's verdicts have no validated correlation with reality.

### 5.1 The calibration set

A subset of golden-set cases (typically 30-60 cases, varying by suite size) constitutes the calibration set. Each case in the calibration set has been reviewed by a human (typically a domain expert):

- The human reviewer applies the same rubric the judge applies.
- The human's verdict (pass / fail) is captured.
- The judge's verdict (when the judge runs on the case) is captured.
- Per-case: agree or disagree.

### 5.2 The agreement metric

Across the calibration set:

- **Agreement rate.** What percentage of cases does the judge agree with the human?
- **Disagreement rate.** Inverse.
- **Per-direction disagreement.** Judge-too-strict (judge fails what human passes) vs judge-too-lenient (judge passes what human fails).

The target: agreement rate ≥ 85% on the calibration set.

Below 85%, the judge is not trustworthy as a gate. The team revises the judge prompt; re-runs against the calibration set; iterates.

### 5.3 The calibration iteration

When the judge disagrees with human review:

1. **Read the judge's justification.** Did the judge see the case differently?
2. **Read the human's reasoning.** What did the human see that the judge missed (or vice versa)?
3. **Determine the source of disagreement.**
   - Judge prompt is missing a relevant rubric criterion → update the prompt.
   - Judge model is missing important context → expand the prompt context.
   - Judge is correctly applying the rubric but the human applied a different standard → update the rubric documentation; train reviewers on the standard.
4. **Apply the fix.** Re-run; re-evaluate.

The iteration is open-ended until the calibration set agreement is ≥ 85%.

### 5.4 The reviewer pool

Calibration depends on human reviewers. For Meridian:

- Clinical golden set: 3-clinician panel; majority vote on disagreement.
- Drug-interaction subset: 2-pharmacist panel; majority vote.
- Conversational subset: 2-engineer + 1-clinician panel.
- Refusal subset: 1-clinician + 1-compliance reviewer.

The reviewer pool size matters: too few and the calibration captures one person's opinion; too many and coordination is expensive.

### 5.5 The calibration set refresh

The calibration set is itself maintained:

- Quarterly: ~30% of calibration cases replaced with new cases drawn from recent golden-set additions.
- Keeps the calibration set representative of current system behavior.
- Prevents calibration from becoming a fixed historical baseline.

### 5.6 The calibration documentation

Calibration results are documented:

- Date of calibration.
- Judge prompt version, judge model version.
- Calibration set ID.
- Agreement rate (overall, per-criterion, per-direction).
- Reviewer notes (significant disagreements, decisions made).

Documentation supports:
- Trend tracking (is calibration drifting over time?).
- Audit (compliance reviewers can see the calibration discipline is in place).
- Onboarding (new engineers can read the calibration history).

---

## 6. Drift detection

The judge can drift. Detection is its own discipline.

### 6.1 Drift sources

- **Judge model version change.** Provider releases a new version behind an alias; the judge's verdicts shift.
- **Judge prompt change.** Even an intended change can produce unintended drift on cases not in the calibration set.
- **System-under-test evolution.** The production system's outputs evolve (new prompt, new model, new corpus); the judge's prompt assumptions may no longer hold.
- **Calibration set drift.** If the calibration set is not refreshed, it becomes a fixed historical baseline; the judge may be correctly calibrated on the old set and miscalibrated on current behavior.

### 6.2 Detection patterns

- **Quarterly recalibration.** Re-run the judge against the (refreshed) calibration set; compare to human review; flag agreement drops.
- **Production sampling.** Run the judge on a fresh sample of production interactions; review a random subset manually; check if the judge's pass/fail matches.
- **Trend monitoring.** Aggregate judge-pass-rate over time. A sudden shift correlates with a calibration drift event.

### 6.3 The drift alert

When calibration agreement drops below 85% in a quarterly recalibration:

- The judge is no longer trustworthy as a gate.
- The eval gate is paused (or the verdicts are treated as advisory rather than blocking) until recalibration completes.
- Investigation: what changed? (Judge model? System under test? Workload composition?)
- Recalibration: update the judge prompt; re-run against the calibration set; iterate to ≥ 85%.

### 6.4 The drift incident

A drift event is an incident. The team:

- Pauses the eval gate.
- Documents the drift (when detected, what changed).
- Recalibrates.
- Resumes the gate after agreement is restored.

Treating drift as a Sev-3 / Sev-4 incident keeps the discipline strong. Without it, drift accumulates and the eval becomes unreliable.

---

## 7. Specialized judges

Different rubric criteria may benefit from specialized judges.

### 7.1 The pattern

Instead of one judge that scores all criteria, use multiple judges each specialized for one criterion:

- One judge for factual correctness.
- One judge for citation accuracy.
- One judge for faithfulness.
- One judge for refusal correctness.

Each judge has a focused prompt; calibration is per-judge; the overall verdict is the composition (pass if all sub-judges pass, fail otherwise).

### 7.2 When to use

Specialized judges are right when:
- One rubric criterion requires significantly different reasoning than others.
- The combined-judge prompt is too long and verdict quality suffers.
- One criterion needs a different model or different temperature.

For Meridian clinical eval: a single composite judge handles most criteria; a specialized judge for clinical-claim-accuracy uses a clinical-fine-tuned model.

### 7.3 The cost trade-off

Specialized judges multiply the per-case cost (each case is judged N times rather than 1). The trade-off is justified when the quality improvement is meaningful.

### 7.4 The orchestration

When multiple judges run per case, the orchestration:
- Each judge runs in parallel (when independent).
- Verdicts aggregated by the composition rule.
- A single aggregate verdict is reported.

The judge orchestration is implementation, not architectural; the calibration is per-sub-judge.

---

## 8. Integration patterns

The judge integrates with multiple eval components.

### 8.1 The eval runner

Per [eval-engineering-playbook.md](./eval-engineering-playbook.md), the eval runner:

1. Loads cases.
2. For each case: invokes the production system (or replays a previously-captured response).
3. Sends the question, expected answer, produced answer, rubric to the judge.
4. Receives the judge's verdict; records.
5. Aggregates pass/fail across cases; reports.

### 8.2 The eval gate

Per [eval-gate-architecture.md](./eval-gate-architecture.md), the gate runs the suite on every PR. The judge is invoked once per case; verdicts gate the merge.

### 8.3 The online judge

Per [online-eval-and-feedback.md](./online-eval-and-feedback.md), a sampled subset of production interactions is run through the judge in reference-free mode (no expected answer). Verdicts feed the production quality SLI.

### 8.4 The observability

Judge verdicts are emitted as trace attributes:

- `ai.eval.judge.applied`: True if this interaction was judged.
- `ai.eval.judge.verdict`: pass / fail.
- `ai.eval.judge.criteria_failed`: which criteria failed.
- `ai.eval.judge.model_version`: judge model.
- `ai.eval.judge.prompt_version`: judge prompt.

The trace allows correlating quality signal with production behavior.

### 8.5 The cost telemetry

Judge calls are LLM calls; they go through the standard LLM-call wrapper per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md); cost is attributed to the eval-runner feature.

The eval cost is itself a tracked line in the FinOps dashboard.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The judge architecture

Meridian's Care Coordinator eval uses a composite judge:

- **Reference-based judge** for the offline clinical golden set: scores against expected_answer + required_claims.
- **Reference-free judge** for online sampling on production traffic: scores against the rubric only.
- **Specialized clinical-claim-accuracy judge** for high-stakes clinical claims: dedicated Opus prompt with extra clinical context.

The composite verdict: pass iff all sub-judges pass.

### 9.2 The judge model selection

- Production system: Opus 4.7 for clinical reasoning; Sonnet for drafting.
- Judge model: Opus 4.7 (same model as production for clinical) with a different (more capable judge) prompt.
- Cross-provider judge experiment in 2026-Q1: tested GPT-5 as a cross-provider judge; agreement with clinical reviewers was 81% vs Opus-judge's 89%. Stayed with Opus-judge.

The judge model version is pinned in the eval-suite release manifest. Anthropic Opus version changes trigger a recalibration event.

### 9.3 The calibration discipline

The clinical calibration set:
- 40 cases drawn from the clinical golden set, balanced across case classes.
- 3-clinician review panel; majority vote on disagreement.
- Quarterly recalibration.

Recent calibration cycles:
- 2026-Q1: agreement 89%; one judge-prompt revision (the "appropriate" criterion needed sharper definition).
- 2026-Q2: agreement 91%; no revisions needed.

Below 85% triggers a Sev-3 calibration-drift incident.

### 9.4 The judge prompt versioning

The judge prompts (one per sub-judge) are versioned per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md). They live alongside production prompts but in an `eval/` subdirectory.

Recent versions:
- `clinical_composite_judge@2.1.0`: current production.
- `clinical_claim_specialist_judge@1.3.0`: current production.
- `clinical_composite_judge@2.0.0`: previous; rollback target.

Eval-suite release manifests pin the judge prompt versions.

### 9.5 The judge call observability

Every judge call is in the trace:

```
span: eval_judge_call
  attributes:
    ai.eval.case_id: CLIN-0042
    ai.eval.judge.model: claude-opus-4-7
    ai.eval.judge.model_version: 2026-04-12
    ai.eval.judge.prompt_version: clinical_composite_judge@2.1.0
    ai.eval.judge.verdict: pass
    ai.eval.judge.criteria_results:
      factually_correct: pass
      complete: pass
      cites_correctly: pass
      appropriate: pass
    ai.cost.usd: 0.012
    ai.eval.judge.latency_ms: 1840
```

The trace supports drift investigation and per-case debugging.

### 9.6 The cost line

The clinical eval suite (200 cases × ~3 judge calls per case × ~$0.012 per call) costs ~$7 per full run. With nightly + release-candidate + per-PR-subset:

- Nightly: ~$7.
- Release candidate: ~$7.
- Per-PR subset (30 cases): ~$1.
- Monthly total: ~$300 in judge calls.

The cost is justified; a single missed quality regression that the eval would have caught easily exceeds the monthly judge cost.

### 9.7 The 2026-Q1 calibration-drift incident

In 2026-Q1, scheduled quarterly calibration showed agreement dropping from 89% to 79%. Investigation:

- Judge prompt version: unchanged.
- Judge model version: had been auto-resolved from `claude-opus-latest` to a new version 2 weeks prior.
- The new judge model interpreted the "appropriate" criterion more strictly; was failing cases the clinical panel rated as acceptable.

The remediation:
- Judge model pinned to the previous specific version (full version string).
- Judge prompt updated with more explicit "appropriate" criterion definition.
- Calibration re-run; agreement restored to 89%.
- Aliases banned from the judge model registry.

The incident motivated the broader model-alias ban discussed in [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).

### 9.8 The platform discipline

- Judge prompts as code; versioned; PR-reviewed.
- Judge model pinned; aliases rejected.
- Quarterly calibration scheduled.
- Calibration agreement < 85% is a Sev-3 incident.
- Drift detection via quarterly recalibration; trend dashboards.

---

## 10. Anti-patterns

### 10.1 "Same-tier judge"

The judge runs on the same model as production. The judge agrees with production's mistakes. False-negative rate is high; quality issues go undetected.

**Corrective.** Judge on a more capable tier per section 3.1.

### 10.2 "Uncalibrated judge"

The judge is in production use; agreement with human review has never been measured. The judge's verdicts are trusted without basis.

**Corrective.** Calibration discipline per section 5; ≥ 85% agreement before judge gates anything.

### 10.3 "Judge prompt is informal"

The judge prompt was written once; not versioned; not eval-validated; not reviewed.

**Corrective.** Judge prompts as code per section 4.5; versioned; PR-reviewed; eval-validated.

### 10.4 "Drift not detected"

Calibration was done at adoption; not re-run since. The judge has drifted (model version change, prompt rot, workload evolution); the team doesn't know.

**Corrective.** Quarterly recalibration per section 6.

### 10.5 "Judge model uses aliases"

The judge model reference is `claude-opus-latest`. Provider-side version changes shift the judge's behavior silently. Calibration breaks.

**Corrective.** Pin to full version strings; treat judge model changes as calibration events.

### 10.6 "Justifications not captured"

The judge returns pass/fail without justification. Disagreement investigation requires re-running the judge with debug logging.

**Corrective.** Per-verdict justification per section 4.4.

### 10.7 "Output not structured"

The judge returns prose. Aggregation requires parsing; parsing fails on formatting variations.

**Corrective.** Structured JSON output per section 4.3; validated post-judgment.

### 10.8 "Calibration set frozen"

The calibration set was built at adoption; never refreshed. The set is no longer representative; calibration is technically measured but not on current behavior.

**Corrective.** Quarterly partial-refresh per section 5.5.

---

## 11. Findings (sprint-assignable)

### JUDGE-001 — Severity: Critical
**Finding.** LLM-as-judge runs on the same model tier as production; same-tier under-detection.
**Recommendation.** Judge on more capable tier per section 3.1; re-validate.
**Owner.** ai-platform-eng, sprint N+1.

### JUDGE-002 — Severity: Critical
**Finding.** Judge has not been calibrated against human review; verdicts are trusted without basis.
**Recommendation.** Calibration per section 5; agreement ≥ 85% before judge gates.
**Owner.** ai-platform-eng + domain experts, sprint N+1.

### JUDGE-003 — Severity: Critical
**Finding.** Judge model is unpinned (alias); silent version drift has caused calibration breaks.
**Recommendation.** Pin judge model to full version; treat changes as calibration events.
**Owner.** ai-platform-eng, sprint N+1.

### JUDGE-004 — Severity: High
**Finding.** Judge prompt is informal; not versioned; ad-hoc changes ship without re-validation.
**Recommendation.** Judge prompts as code per section 4.5; PR-reviewed; eval-validated.
**Owner.** ai-platform-eng, sprint N+2.

### JUDGE-005 — Severity: High
**Finding.** Judge output is not structured; aggregation requires fragile parsing.
**Recommendation.** Structured JSON output per section 4.3; validated post-judgment.
**Owner.** ai-platform-eng, sprint N+2.

### JUDGE-006 — Severity: High
**Finding.** Judge drift is not detected; calibration was done once and never re-validated.
**Recommendation.** Quarterly recalibration per section 6.2.
**Owner.** ai-platform-eng + domain experts, sprint N+2.

### JUDGE-007 — Severity: High
**Finding.** Justification field is not captured; disagreement investigation is opaque.
**Recommendation.** Per-verdict justification per section 4.4.
**Owner.** ai-platform-eng, sprint N+2.

### JUDGE-008 — Severity: High
**Finding.** Calibration set is frozen; no refresh since adoption.
**Recommendation.** Quarterly partial-refresh per section 5.5.
**Owner.** ai-platform-eng, sprint N+3.

### JUDGE-009 — Severity: Medium
**Finding.** Calibration documentation is absent; agreement history is not tracked.
**Recommendation.** Calibration documentation per section 5.6; trend tracking.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### JUDGE-010 — Severity: Medium
**Finding.** Drift incidents are not formalized; calibration breaks go unaddressed.
**Recommendation.** Drift incident protocol per section 6.4; pause gate; recalibrate.
**Owner.** ai-platform-eng + sre, sprint N+3.

### JUDGE-011 — Severity: Medium
**Finding.** Specialized judges are not considered; single composite judge handles all criteria with sub-optimal results.
**Recommendation.** Per-criterion specialized judges where rubric criteria differ significantly per section 7.
**Owner.** ai-platform-eng, sprint N+3.

### JUDGE-012 — Severity: Medium
**Finding.** Reviewer pool is a single domain expert; calibration captures one person's opinion.
**Recommendation.** 2-3 reviewer panel; majority vote per section 5.4.
**Owner.** ai-platform-eng + domain expertise team, sprint N+3.

### JUDGE-013 — Severity: Medium
**Finding.** Judge cost is not separately tracked; eval cost is invisible in FinOps.
**Recommendation.** Per-feature cost attribution; eval as its own feature.
**Owner.** ai-platform-eng + finops, sprint N+4.

### JUDGE-014 — Severity: Medium
**Finding.** Judge verdicts are not in traces; cross-system correlation of quality with production behavior is manual.
**Recommendation.** Judge attributes on traces per section 8.4.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### JUDGE-015 — Severity: Medium
**Finding.** Cross-provider judge has not been considered; shared-bias risk with same-provider judge is unexamined.
**Recommendation.** Experiment with cross-provider judge; compare calibration.
**Owner.** ai-platform-eng, sprint N+4.

### JUDGE-016 — Severity: Low
**Finding.** Pairwise comparison judge is not used for model comparison; system-comparison evaluations are ad-hoc.
**Recommendation.** Pairwise judge per section 2.3 for explicit comparison evaluations.
**Owner.** ai-platform-eng, sprint N+4.

### JUDGE-017 — Severity: Low
**Finding.** Online judge runs on production but in same configuration as offline judge; sample rate is uncalibrated.
**Recommendation.** Online-specific sample rate calibration per [online-eval-and-feedback.md](./online-eval-and-feedback.md).
**Owner.** ai-platform-eng, sprint N+5.

### JUDGE-018 — Severity: Low
**Finding.** Judge prompt documentation is thin; new engineers do not understand the judge architecture.
**Recommendation.** Documentation alongside prompt artifacts; include in onboarding.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team starting from "we want LLM-as-judge but don't have one":

- [ ] **Sprint 0 — design.** Choose architecture (reference-based for offline; reference-free for online). Pick judge model (more capable than production).
- [ ] **Sprint 1 — first judge.** Write the judge prompt. Pin judge model version. Run against the golden set; capture verdicts.
- [ ] **Sprint 1 — calibration set.** Build the 30-60 case calibration set. Domain experts review each.
- [ ] **Sprint 2 — calibration.** Compare judge verdicts to human reviews. Iterate the judge prompt until agreement ≥ 85%.
- [ ] **Sprint 2 — structured output.** JSON verdict format; validated parsing.
- [ ] **Sprint 3 — observability.** Judge verdicts in traces; judge cost in FinOps.
- [ ] **Sprint 3 — drift detection.** Schedule quarterly recalibration; document agreement history.
- [ ] **Sprint 4 — specialized judges.** Add per-criterion specialized judges where rubric differs significantly.
- [ ] **Sprint 4 — online judge.** Sampled production runs; SLI for quality.
- [ ] **Sprint 5 — drift incident protocol.** Document; rehearse.
- [ ] **Ongoing — discipline.** Quarterly calibration; judge prompt versioning; model-version pinning.

A team that completes this sequence has an LLM-as-judge that the team can trust. A team that skips calibration has a judge that produces verdicts the team cannot use to make decisions.

---

## 13. References

- This repo: [eval-engineering/eval-engineering-playbook.md](./eval-engineering-playbook.md) — the broader practice this is the depth on (section 4).
- This repo: [eval-engineering/golden-set-design.md](./golden-set-design.md) — what the judge scores against.
- This repo: [eval-engineering/regression-eval-suites.md](./regression-eval-suites.md) — uses the judge.
- This repo: [eval-engineering/online-eval-and-feedback.md](./online-eval-and-feedback.md) — the judge in production sampling.
- This repo: [eval-engineering/eval-gate-architecture.md](./eval-gate-architecture.md) — the CI integration.
- This repo: [eval-engineering/eval-of-rag.md](./eval-of-rag.md) — RAG-specific judge patterns.
- This repo: [prompt-engineering/prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md) — judge prompts as code.
- This repo: [model-lifecycle/model-registry.md](../model-lifecycle/model-registry.md) — judge model registration.
- This repo: [cicd-and-eval-gates/model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) — judge model pinning.
- Anthropic, OpenAI documentation on judge / evaluation patterns.
- Research: AlpacaEval, MT-Bench, G-Eval, LLM-as-a-judge survey literature.
