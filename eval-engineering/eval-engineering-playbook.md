# Eval Engineering Playbook

> **Audience.** Engineers and tech leads building the eval practice for an AI feature or platform from scratch, or fixing a stalled one. **Scope.** The engineering practice of evals — building, running, calibrating, gating, and operating them. Not the methodology of eval research, not the academic benchmark canon, not the architectural choice of *which patterns to evaluate against* (the architecture sibling repo owns that). **Worked client.** Meridian Health, the fictional regulated healthcare SaaS used across the sibling reference architecture repos.

---

## 1. Why this document exists

Evals are the load-bearing engineering practice for AI systems. Without them a team cannot safely deploy, roll back, swap models, refactor prompts, change retrieval, or know whether a fix actually fixed anything. With them, all of the above become tractable engineering operations on the same scale as a database migration. The gap between teams that ship AI features confidently and teams that ship them anxiously is almost entirely the eval gap.

Most teams know this. They still defer eval engineering because the upfront investment looks heavy and the payback is invisible until something breaks. So this playbook is structured for *incremental adoption*. The first three sections take a team from zero to a useful eval in a sprint. The middle sections add LLM-as-judge with calibration, regression suites, and CI gating. The last sections cover the operational depth a team needs once the practice is real: drift management, judge recalibration, contamination prevention, agent and RAG eval extensions, anti-patterns, and the worked Meridian Care Coordinator example end-to-end.

A team that adopts this playbook in order will land:

1. **Sprint 1.** A 20-case golden set with binary pass/fail and a manual review process.
2. **Sprint 2.** An LLM-as-judge harness calibrated against the manual reviews.
3. **Sprint 3.** Eval gate in CI blocking PRs on regression.
4. **Sprint 4–6.** Regression coverage from production incidents, online eval signals, judge drift monitoring.
5. **Sprint 7+.** Specialized suites (RAG-stage eval, agent trajectory eval, adversarial / refusal coverage).

Each step is independently useful. None of them require the next step to be valuable.

---

## 2. The eval taxonomy

Evals come in distinguishable forms; conflating them is one of the most common reasons teams get stuck. Use these names consistently:

- **Offline evals.** Run on a fixed dataset (the golden set or a regression suite), out of band from production traffic. Deterministic, repeatable, used for gates and trend reporting.
- **Online evals.** Run on production traffic — sampled judge runs over real interactions, explicit user feedback (thumbs / ratings), implicit user feedback (retry, edit, abandonment). Noisy, real, used for drift detection and golden-set growth.
- **Reference-based.** The eval has a known correct answer (or accepted set of correct answers); scoring is against the reference.
- **Reference-free / rubric-based.** No single correct answer; the eval scores against a rubric (factual accuracy, tone, completeness, refusal-correctness). LLM-as-judge is the typical implementation.
- **Outcome eval.** Only the final answer is scored.
- **Trajectory / step eval.** Each intermediate step (tool call, retrieval, sub-agent invocation) is scored.
- **Judge eval.** A separate eval of the judge itself — does the judge agree with human raters? Calibration is the practice of keeping the judge eval green.

A mature eval practice runs all of these in some form. A starting eval practice should be one offline reference-based outcome eval with a manual review process and nothing else.

---

## 3. Sprint 1 — Build a 20-case golden set

The single highest-leverage move in any team's eval journey. Most teams skip this step in favor of building infrastructure first; the team that has 20 cases and runs them manually is further along than the team that has a fancy eval platform and no cases.

### 3.1 Case selection

Twenty cases is enough to be useful, small enough to be reviewable by humans every week, and tight enough to force you to choose the cases that *actually matter*. The selection criteria:

- **Five representative cases.** The most common shape of user interaction. If the system is a Q&A bot, these are the most-frequent question shapes.
- **Five edge-case cases.** Adversarial inputs (ambiguous question, missing context, contradictory premise), uncommon but important shapes, the long tail.
- **Five known-failure cases.** Real cases that have produced wrong answers (in dogfooding, in pilots, or in early production). If you don't have any yet, derive them by asking domain experts "what is the question shape we are most likely to get wrong?"
- **Five high-stakes cases.** The cases where a wrong answer is the most expensive — regulated content, customer-visible commitments, anything you do not want a screenshot of in a Slack channel.

Twenty cases is a starting point. Plan to grow to 100–300 in a quarter and 500–1000 within a year. But ship the 20 first.

### 3.2 Case structure

Each golden-set case has five fields:

```yaml
id: GOLD-0014
question: "What is the post-discharge follow-up protocol for a CHF patient on the new pathway?"
expected_answer: |
  The post-discharge follow-up for a CHF patient on the new pathway is a 7-day
  nursing check-in and a 14-day clinician visit. The protocol cites the
  AHA 2024 Heart Failure Discharge Bundle and Mercy Cleveland Protocol HF-22.
expected_sources:
  - "AHA 2024 Heart Failure Discharge Bundle, section 3.2"
  - "Mercy Cleveland Protocol HF-22"
case_class: "clinical-protocol"
notes: "First-turn lookup; tests both global guideline and tenant-protocol retrieval."
```

Three things to notice:

- **`expected_answer` is a freeform reference.** Not the exact string the model must produce — that is too brittle. It is the answer a human reviewer would accept as correct.
- **`expected_sources` are explicit.** This lets retrieval recall be scored separately from answer correctness, which is the single most useful diagnostic distinction in RAG systems.
- **`case_class` enables sub-suite analysis.** Aggregate scores per class to see *where* quality is good or bad.

### 3.3 Scoring rubric

Define what "pass" means before running the eval. Binary pass/fail is the right starting point — scalar scoring is more nuanced and almost always less informative early on.

A case passes if:

1. The answer is **factually correct** — every claim it makes is supportable by the expected sources (or by other available correct sources).
2. The answer is **complete** — it addresses the actual question, not just a related one.
3. The answer **cites correctly** — claims that should be cited are cited, citations point to real sources, citations support the claim made.
4. The answer is **appropriate** — it does not over-claim, does not hallucinate a confidence level, does not skip refusal in cases where refusal is warranted.

Fails on any one = case fails.

### 3.4 Run the eval

For the first run, *run it manually*. Send each question through the system, collect each answer, score each against the rubric. Manually. This is mandatory — you are calibrating your own judgment, building intuition for the system's failure modes, and producing the data that the LLM-as-judge in Sprint 2 will be calibrated against.

Track pass rate per case class. Track *why* fails fail. After this first manual pass, you will have:

- A pass-rate number (the first eval result of the practice).
- A taxonomy of failure modes (the diagnostic surface).
- A view of which case classes need work.

This is real, actionable evaluation. It cost about three engineer-days. Most of the rest of this playbook is about scaling, automating, and embedding this practice — but the practice is *here*, on day one.

---

## 4. Sprint 2 — LLM-as-judge with calibration

Manual review does not scale beyond a few hundred cases or a few iterations per week. The standard production pattern is LLM-as-judge: a separate (typically more capable) model scores each answer against the rubric. Done correctly, this is the workhorse of eval automation. Done incorrectly, it is a noise generator.

### 4.1 The judge prompt

The judge sees the question, the expected answer (and expected sources), the produced answer, and the rubric. It returns a structured verdict.

```
You are evaluating an answer to a clinical question.

QUESTION:
{question}

EXPECTED ANSWER (for reference; correct answers may differ in wording):
{expected_answer}

EXPECTED SOURCES (the answer should cite content equivalent to one or more of these):
{expected_sources}

PRODUCED ANSWER:
{produced_answer}

RUBRIC:
1. Factually correct: every claim is supportable by the expected sources or other
   available correct sources.
2. Complete: addresses the actual question, not a related one.
3. Cites correctly: claims requiring citation are cited; citations point to real
   sources; citations support the cited claim.
4. Appropriate: does not over-claim, does not hallucinate confidence, refuses
   when refusal is warranted.

Score each criterion as pass/fail and provide a one-sentence justification.
Then return overall PASS or FAIL. The answer FAILS if any one criterion fails.

Return JSON:
{
  "factually_correct": {"verdict": "pass|fail", "justification": "..."},
  "complete":          {"verdict": "pass|fail", "justification": "..."},
  "cites_correctly":   {"verdict": "pass|fail", "justification": "..."},
  "appropriate":       {"verdict": "pass|fail", "justification": "..."},
  "overall":           "pass|fail"
}
```

The structured output is non-negotiable. A judge that returns prose has nothing to aggregate.

### 4.2 Judge model selection

The judge should be more capable than the model under test. If the system uses Sonnet-class for production, the judge should be Opus-class. Same-tier or weaker judges silently agree with the system's mistakes.

Pin the judge model version. A judge that drifts because the SDK auto-upgraded the underlying model produces eval results that are not comparable across time, which is the worst possible property for an eval.

### 4.3 Calibration

This is the step most teams skip and then never trust their eval. The discipline: take the 20 golden-set cases that were manually reviewed in Sprint 1. Run the judge over them. Compare the judge's verdict to the manual verdict, case by case.

Three outcomes per case:
- **Agreement.** Manual and judge both pass, or both fail. Good.
- **Judge over-strict.** Manual passed, judge failed. Look at the justification; the judge is usually flagging a real defect that the manual reviewer was tolerant of. Decide whether to update the manual standard or the judge prompt.
- **Judge over-lenient.** Manual failed, judge passed. Look at the justification; the judge missed something. Update the judge prompt to be specific about the failure mode.

Repeat: re-run after every judge-prompt change, compare to the manual reviews, iterate. The goal is *agreement rate above ~85% on the calibration set*. Below that, the judge is not trustworthy as a gate.

Recalibrate quarterly and after any judge model change. Recalibration is the eval-engineering equivalent of "the meter that measures the thing must itself be measured."

### 4.4 What "fail" means at this point

Once judge calibration is above 85%, the judge can be the primary eval signal — running automatically, fast, cheap enough to run on every PR. The manual review process continues at a reduced cadence (10–20% sample, weekly) as the calibration ground truth.

---

## 5. Sprint 3 — Eval gate in CI

An eval that does not gate anything is a dashboard. A dashboard does not prevent regressions. The most important step in the eval engineering journey is making the eval *block deployment when it regresses*.

### 5.1 Fast suite vs full suite

The full suite (eventually hundreds of cases) is too slow and expensive to run on every PR. Split it:

- **Fast suite.** ~25–40 cases that exercise representative class coverage. Runs in under 10 minutes. Gates every PR.
- **Full suite.** All cases. Runs nightly on `main` and on every release candidate. Trend data and the canonical quality measurement.

Selecting the fast suite is itself an engineering decision. The fast suite must cover every case class (representative + edge + known-failure + high-stakes) proportionally, must include cases that frequently regress, and must be refreshed when production failure modes shift. Treat the fast-suite composition as a versioned artifact.

### 5.2 Pipeline architecture

```
PR open / push to PR branch
   │
   ▼
Lint (prompt structure, schema validity, code lint)
   │
   ▼
Unit tests
   │
   ▼
Fast eval suite — block merge on regression (pass-rate drop > N%)
   │
   ▼
Code review + approval
   │
   ▼
Merge to main
   │
   ▼
Full eval suite (nightly + on release candidate)
   │
   ▼
Canary deploy + online eval monitoring
   │
   ▼
Promote to 100% or roll back
```

### 5.3 Threshold setting

A fast suite of 30 cases with binary pass/fail has natural noise — a single flipped case is a ~3-point movement in pass rate. Setting "block on any drop" produces alert fatigue and false-positive blocks; setting "block on > 10 points" misses real regressions on a smaller suite.

A reasonable starting threshold: *block if pass-rate drops below the trailing 7-day baseline minus 5 percentage points, OR if the absolute pass rate drops below an explicit floor (typically 85% for general workloads, 95% for high-stakes workloads)*. Tune over the first month based on observed noise.

For high-stakes case classes (Meridian's drug-interaction subset, say), the threshold is tighter and overrides the general threshold — a single regression on those cases blocks merge regardless of overall pass rate.

### 5.4 The override

An eval gate without an override is a brittle gate. There are legitimate cases where a regression is intentional (a prompt change deliberately tightens refusal behavior, dropping pass rate on cases that were *previously* too lenient). The override pattern: a labeled commit (`[eval-override: reason]`) plus reviewer approval bypasses the gate, and the override is logged for post-deploy review.

The override discipline matters more than the override mechanism. If overrides become routine, the gate is decoration. Track override usage; high override rates are a signal that the eval, not the system, needs work.

---

## 6. Sprint 4 — Regression suites from real incidents

Every production quality bug becomes a permanent eval case. This is how the eval suite stops being a static artifact and starts being a living record of the system's known failure modes.

The pattern: when a bug is filed against the AI system — *the model gave a wrong answer to question X* — the fix lifecycle adds two steps:

1. **Reproduce and capture.** Re-run the question, capture the wrong answer. This becomes a case.
2. **Add to regression suite.** The case enters a `regression/` subdirectory of the eval suite with the original wrong answer noted (for context) and the correct answer as `expected_answer`.

Within a quarter of operating this practice, the regression suite is the single most valuable subset of the eval suite — it is composed entirely of cases that *have already failed in production at least once*. A passing regression suite is strong evidence that previously-fixed bugs have not regressed.

The discipline that makes this work: the bug ticket cannot close until the regression case is added. This is a process gate, not an eval-engineering gate, but it is the eval engineer's job to make it cheap to comply with — a one-command "add this question + the corrected answer to the regression suite" tool removes the friction that otherwise kills the practice.

---

## 7. Sprint 5 — Online evals and feedback loops

The offline suite tests what you knew to test for. The online signals tell you what you did not know to test for. Both are necessary; neither is sufficient alone.

### 7.1 Explicit feedback

Thumbs-up / thumbs-down, ratings, free-text feedback. Aggregated as a quality signal but more useful as a *case-mining surface* — thumbs-down responses with free-text are the highest-quality source of new golden-set cases.

Build the pipeline once: every thumbs-down + free-text is reviewed weekly by the eval team, candidate cases are extracted, and the highest-leverage candidates are added to the golden set. Most thumbs-down responses do not become golden-set cases — they are signal noise — but enough of them do that the practice pays back.

### 7.2 Implicit feedback

Retries, edits, abandonment, escalation to human. Each of these is a "the answer was not quite right" signal that does not require the user to do anything explicit. The engineering work is in instrumenting them — instrumentation must be in the application layer, not derived after the fact from logs.

Useful implicit signals for AI features:

- **Retry rate.** User asked the same or near-same question again within a short window. High retry rate on a feature is a quality signal.
- **Edit rate.** For drafting workflows, the rate at which users edit the draft before sending it. Some edit is normal; trending higher is a regression signal.
- **Abandonment rate.** User started but did not complete the interaction. Distinguishing "abandoned because wrong answer" from "abandoned because interrupted" is hard but worth attempting via session shape analysis.
- **Escalation rate.** For systems with a human-escalation path, the rate of escalation is a near-direct quality signal — the system itself is saying "I cannot handle this."

### 7.3 Sampled online judge runs

A randomly sampled subset of production interactions is run through the LLM-as-judge (with the actual produced answer; no expected-answer reference is available, so the judge falls back to reference-free rubric scoring of factual claims, citation accuracy, format, and refusal-appropriateness). The judge-pass-rate becomes a real-time SLI of system quality.

Sampling rate trade-off: 100% sampling is the most informative but doubles the AI bill on the system. 5–10% sampling is the common starting point. Sample rate is class-stratified — high-stakes case classes are sampled more heavily.

### 7.4 The feedback loop

```
production interactions → sampled judge runs → judge-pass-rate dashboard
                       → thumbs/ratings ──────→ feedback inbox → weekly triage
                       → implicit signals ────→ retry/edit/abandonment dashboards
                                                       │
                                                       ▼
                                         high-signal cases enter the golden set
                                                       │
                                                       ▼
                                            next offline eval cycle
```

The loop is the practice. The dashboards alone, without the pipeline that promotes signal to cases, do not produce learning.

---

## 8. Judge drift monitoring and recalibration

A judge calibrated in March may not be calibrated in July. Causes of drift:

- **Judge model version changed.** Either explicitly (you upgraded) or implicitly (the SDK auto-upgraded). Pinning prevents implicit.
- **The system under test changed enough that the judge prompt's assumptions no longer hold.** The judge prompt that worked when the system answered in 200 words may be over-strict when the system now answers in 50 words.
- **The rubric tightened.** Decisions made over time to consider previously-acceptable patterns as failures need to propagate into the judge.
- **Calibration set drift.** The golden set grew, but the calibration set (the human-reviewed subset) did not, so calibration is measured on increasingly-old cases.

Operational practice:

- **Re-run calibration quarterly.** Take the current calibration-set human reviews, re-run the judge, compare. Update the judge prompt as needed.
- **Page on judge agreement drop.** If quarterly calibration shows judge-human agreement dropped from 89% to 78%, the judge is no longer trustworthy as a gate. Treat as an incident; pause the eval gate until recalibrated.
- **Refresh the calibration set.** Every quarter, replace ~30% of the calibration-set cases with new ones drawn from recent golden-set additions. Keeps the calibration set representative of current system behavior.

This is the eval-of-the-eval. Without it, the eval slowly stops measuring what it thinks it measures.

---

## 9. Contamination prevention

Eval contamination is the silent killer. It happens when cases from the eval set leak into something the system has trained on, been fine-tuned on, been few-shotted with, or has retrieved. When it happens, the eval score becomes meaningless — the system is being tested on its memorization, not its capability.

Sources of contamination and the engineered preventions:

- **Fine-tune data overlap.** Eval cases were used as training examples. Prevention: hash-based separation — every fine-tune dataset is processed through a deduplication step against the eval set; matches are removed from the fine-tune set.
- **Few-shot example overlap.** Eval cases were used as in-prompt examples. Prevention: separate eval and example pools; never sample from the eval set.
- **Retrieval corpus overlap.** Eval cases were ingested into the retrieval corpus. Prevention: tag eval cases with a `do_not_ingest` flag; the ingestion pipeline rejects flagged content.
- **Online-eval / golden-set bleed.** A case promoted from online sampling was already on the golden set. Prevention: deduplicate on promotion; track lineage of each case.
- **Cross-tenant or cross-feature leakage.** A multi-feature system shares evals across features; a feature's evaluation accidentally tests on data the system has seen in another context. Prevention: per-feature contamination audits.

The audit cadence: quarterly contamination check — sample 30 eval cases, search the training / fine-tune / retrieval data for near-matches, report. A low contamination rate (under 1–2%) is typical and not actionable; a sudden spike is.

---

## 10. RAG-specific eval engineering

Generic answer-quality eval treats the system as a black box. For RAG systems this is a missed opportunity — the diagnostic value of separating *retrieval* failures from *generation* failures is enormous, because the fixes are entirely different (retriever tuning vs prompt tuning vs context-window tuning).

The discipline: every RAG eval case captures `expected_sources` (which documents / chunks should be retrieved). The eval pipeline then scores two things separately:

- **Retrieval recall.** Did the expected source(s) appear in the top-K retrieved chunks?
- **Answer correctness given the actual retrieved chunks.** Was the answer correct, given what was actually retrieved? (This is the generation-stage quality, isolated from retrieval quality.)

When the overall case fails, the breakdown tells you what to fix:

| Retrieval recall | Answer correct given retrieved | Diagnosis |
|---|---|---|
| Pass | Pass | The case passed. |
| Pass | Fail | Generation problem (prompt, model, context use). The retrieval did its job. |
| Fail | (Pass or Fail) | Retrieval problem. No amount of prompt-tuning will fix this case. |

This is one of the highest-leverage moves in RAG eval engineering. Without it, every failure looks the same and "improve quality" is the only available action.

Additional RAG-specific eval dimensions: **citation accuracy** (the cited source actually contains the claim), **faithfulness** (the answer is grounded in the retrieved content and does not introduce un-retrieved claims), **empty-retrieval rate** (how often does retrieval return nothing useful, and does the system handle that gracefully).

---

## 11. Agent-specific eval engineering

Agent evals are harder than single-call evals along every dimension. The agent's trajectory is non-deterministic — the same question can lead to different tool-call sequences, different retrieval queries, different intermediate states. The eval must accommodate that without becoming so loose that it stops catching regressions.

Three eval shapes for agents, used together:

- **Outcome eval.** Was the final answer correct? Simplest. Necessary but not sufficient — passes a case where the agent reached the right answer by luck after 15 wrong steps.
- **Trajectory eval.** Were the intermediate steps reasonable? Score the tool calls (right tool for the right step), the retrieval queries (likely to retrieve relevant content), the planning decisions. Done by an LLM judge with the trajectory in context.
- **Step-budget eval.** Did the agent complete within its turn / cost budget? Catches the agent that gets the right answer but burns 20 turns to do so.

Cost-aware sampling matters more for agents than for single-call systems. A 300-case agent eval can cost $50–$300 to run. A subset (~30 cases) for the per-PR fast suite; the full suite runs less often.

The regression discipline applies: every observed agent failure (a loop, a wrong tool call, an over-spent budget) becomes a trajectory eval case.

---

## 12. The Meridian Care Coordinator eval surface (worked example)

This is what the eval practice looks like at the destination, applied to the Care Coordinator (see the architecture sibling's [reference-systems/meridian-care-coordinator.md](../../ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md) for system context).

### 12.1 Suites

| Suite | Purpose | Size | Refresh | SLO threshold |
|---|---|---|---|---|
| Clinical golden set | Outcome + retrieval correctness on representative clinical queries | 200 | Quarterly + incident-driven | Answer-correct ≥ 95% |
| Drug-interaction | Outcome correctness on interaction-shaped queries (uses the FDA graph) | 60 | Quarterly + on FDA SPL changes | Answer-correct ≥ 98%, citation ≥ 99% |
| Conversational | Recall + coherence on second-turn-and-later queries | 50 multi-turn | Quarterly | Recall ≥ 85% turn 2+ |
| Side-effect / HITL | Tool-call accuracy, authorization correctness | 40 | Quarterly + on tool changes | Tool accuracy ≥ 99%, authz 100% |
| Refusal / escalation | Refusal-appropriate cases | 30 | Quarterly | Refuses-when-should ≥ 95% |
| Adversarial / injection | Prompt-injection resistance (jointly owned with the ai-security sibling) | 25 | Quarterly | Resistant ≥ 95% |
| Regression | Real production-bug cases | growing (currently 87) | Per-bug-fix | No regressions allowed |

### 12.2 Pipeline integration

- **Per PR.** Fast suite: 30-case stratified sample across clinical, interaction, conversational, side-effect, and regression. ~7 minutes wall-clock. Blocks merge on regression or absolute-floor breach (95% on clinical, 98% on interaction).
- **Nightly on main.** Full suites run. Trend dashboards update. Cost and latency tracked alongside quality.
- **Release candidate.** All offline suites + fresh adversarial subset + retrieval-recall on a fresh production sample. Blocks release on regression.
- **Production.** 10% sampling of clinical interactions through the LLM-as-judge runs every hour; judge-pass-rate is an SLI with an alerting threshold (page on drop > 8 points from trailing 7-day average).

### 12.3 Calibration

Quarterly: 30-case calibration subset is reviewed by a clinician panel (3 reviewers, majority vote on disagreement); judge is re-run against those reviews; agreement rate computed. Recent calibration cycles have shown judge-clinician agreement at 88–92%; below 85% triggers a judge-prompt-revision sprint.

### 12.4 Cost

Full suite cost: ~$45 per run. Nightly + release-candidate + on-demand totals ~$2,500/month. Trade-off accepted: the cost of a regression caught by eval is much higher than the eval bill, and the audit value of full traceable runs is material.

### 12.5 Findings on the Care Coordinator's eval practice (from the most recent review)

Cross-reference with the architecture sibling's [meridian-care-coordinator.md](../../ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md) finding set:

- ARCH-CARE-010 (eval coverage on side-effect / HITL paths): the fast suite has 3 side-effect cases; expanding to 8–12 is in the current sprint.
- ARCH-CARE-015 (adversarial coverage): the 25-case adversarial set has 4 retrieval-injection cases; expanding to 15+ is a cross-team sprint with the ai-security sibling.

---

## 13. Anti-patterns

The seven patterns I see teams adopt that look like evals but do not function as evals.

### 13.1 Eval-as-vibe-check

The team runs a handful of queries through the system, looks at the answers, says "looks good," and ships. No structure, no rubric, no record. The eval has produced exactly zero reusable signal.

**Corrective.** A 20-case golden set with a written rubric. Time investment: three days. This is the entire content of section 3 of this playbook.

### 13.2 Eval-as-spreadsheet

The team has eval cases in a Google Sheet. Cases get added; results get added; nobody knows which version of the system was tested or which prompt version produced which result. The spreadsheet drifts and is eventually abandoned.

**Corrective.** Eval cases as code, in version control, alongside the system. Results as structured runs with explicit `system_version`, `prompt_version`, `model_version`, `dataset_version` attribution.

### 13.3 Eval-as-judge-without-calibration

The team adopted LLM-as-judge but never calibrated it against human review. The judge agrees with itself; the team trusts it; an entire class of failures is silently passed by the judge and shipped.

**Corrective.** The discipline in section 4.3. Calibration is the foundation; without it the judge eval is decoration.

### 13.4 Eval-as-benchmark

The team's "eval suite" is MMLU, HellaSwag, or a similar academic benchmark. The benchmark is unrelated to the team's workload. Model rankings on the benchmark do not predict performance on the team's actual feature.

**Corrective.** Build the workload-specific golden set. Benchmarks are useful for model selection at the broadest level; they are not a substitute for an eval against your own workload.

### 13.5 Eval-as-runs-when-someone-remembers

The eval exists but is run manually, on demand, when someone has a hunch. Most weeks it does not run. Regressions ship between manual runs.

**Corrective.** Automate. CI integration, nightly runs on main, dashboards that anyone on the team can check.

### 13.6 Eval-as-passive-dashboard

The eval runs automatically. The dashboard shows the pass rate. Nobody acts on it; nothing blocks merge or deployment on regression. Regressions ship and the dashboard turns red, and the dashboard staying red becomes the new normal.

**Corrective.** Gate on the eval. The eval is a control surface, not a measurement surface. If a regression cannot block a merge, it is not a gate.

### 13.7 Eval-as-passed-once-and-frozen

The eval suite was built two years ago, has 80 cases, and has not been updated since. The system has evolved; the cases have not. The eval passes consistently because it does not actually test current behavior.

**Corrective.** Continuous curation. Cases added from incidents, from feedback, from new feature areas. Cases removed when they become irrelevant. The eval suite is a living artifact with an owner.

---

## 14. Findings (sprint-assignable)

The canonical eval-engineering findings I write into review documents. Each has an ID (`EVAL-NNN`), severity, finding, recommendation, sprint owner template.

### EVAL-001 — Severity: Critical
**Finding.** No offline eval suite exists for production AI feature.
**Recommendation.** Build the 20-case golden set per section 3; ship in one sprint.
**Owner.** ai-platform-eng, sprint N+1.

### EVAL-002 — Severity: Critical
**Finding.** LLM-as-judge is in use without calibration against human review.
**Recommendation.** Conduct calibration per section 4.3; do not use judge as a gate until agreement ≥ 85%.
**Owner.** ai-platform-eng, sprint N+1.

### EVAL-003 — Severity: High
**Finding.** Eval gate is not connected to CI; regressions can ship.
**Recommendation.** Add fast eval suite as required PR check; block merge on regression per section 5.
**Owner.** ai-platform-eng + platform-eng, sprint N+2.

### EVAL-004 — Severity: High
**Finding.** Judge model version is unpinned; SDK auto-upgrade has caused judge drift incidents.
**Recommendation.** Pin judge model version explicitly in eval-runner config; treat changes as breaking.
**Owner.** ai-platform-eng, sprint N+1.

### EVAL-005 — Severity: High
**Finding.** No regression suite — production bugs are not captured as permanent eval cases.
**Recommendation.** Add the "bug fix requires regression case" process gate; provide a one-command tool to add cases.
**Owner.** ai-platform-eng, sprint N+2.

### EVAL-006 — Severity: High
**Finding.** RAG system eval does not distinguish retrieval failures from generation failures.
**Recommendation.** Capture `expected_sources` per case; score retrieval recall and answer-given-retrieved separately per section 10.
**Owner.** ai-platform-eng, sprint N+2.

### EVAL-007 — Severity: Medium
**Finding.** Golden set has no case-class taxonomy; failure diagnosis cannot identify which classes regressed.
**Recommendation.** Add `case_class` field; aggregate pass rates per class; surface in dashboard.
**Owner.** ai-platform-eng, sprint N+2.

### EVAL-008 — Severity: Medium
**Finding.** No process for golden-set growth from production feedback.
**Recommendation.** Implement weekly feedback-triage; promote high-signal cases to the golden set.
**Owner.** ai-platform-eng, sprint N+3.

### EVAL-009 — Severity: Medium
**Finding.** No contamination audit; eval-case-in-training-data risk is unmeasured.
**Recommendation.** Quarterly contamination audit per section 9; report contamination rate.
**Owner.** ai-platform-eng, sprint N+3.

### EVAL-010 — Severity: Medium
**Finding.** Online judge sampling not implemented; production quality is not observable in real time.
**Recommendation.** Implement sampled online judge runs per section 7.3; surface judge-pass-rate as an SLI.
**Owner.** ai-platform-eng, sprint N+3.

### EVAL-011 — Severity: Medium
**Finding.** Quarterly judge recalibration is not scheduled; calibration could drift undetected.
**Recommendation.** Schedule quarterly recalibration as a recurring sprint task; alert on agreement-rate drops.
**Owner.** ai-platform-eng, sprint N+4.

### EVAL-012 — Severity: Medium
**Finding.** No fast / full suite split; per-PR runs are too slow.
**Recommendation.** Define fast suite (~25–40 cases stratified across classes) per section 5.1; run fast on PR, full nightly.
**Owner.** ai-platform-eng, sprint N+2.

### EVAL-013 — Severity: Medium
**Finding.** Override mechanism for eval gate is undocumented and untracked.
**Recommendation.** Define the override pattern per section 5.4; log overrides; review override usage monthly.
**Owner.** ai-platform-eng, sprint N+3.

### EVAL-014 — Severity: Medium
**Finding.** Adversarial / refusal subset of eval suite is absent or below threshold coverage.
**Recommendation.** Coordinate with security sibling to land adversarial / refusal suite; target ≥ 25 cases.
**Owner.** ai-platform-eng + ai-security, sprint N+3.

### EVAL-015 — Severity: Medium
**Finding.** Agent system uses outcome-only eval; trajectory failures (over-budget, wrong tool) are not measured.
**Recommendation.** Add trajectory eval and step-budget eval per section 11.
**Owner.** ai-platform-eng, sprint N+3.

### EVAL-016 — Severity: Low
**Finding.** Eval pipeline cost is not tracked alongside production cost.
**Recommendation.** Add eval-cost line to FinOps dashboards; budget eval as a fixed engineering cost.
**Owner.** ai-platform-eng, sprint N+4.

### EVAL-017 — Severity: Low
**Finding.** Eval suite owner is unclear; cases accumulate without curation.
**Recommendation.** Assign explicit eval-suite owner; quarterly curation pass to retire stale cases.
**Owner.** ai-platform-eng team lead, sprint N+4.

### EVAL-018 — Severity: Low
**Finding.** Eval cases are not attributed to source (incident, feedback, dogfood, designed); cannot triage representativeness.
**Recommendation.** Add `source` field to each case; report case-source distribution.
**Owner.** ai-platform-eng, sprint N+5.

---

## 15. Adoption sequencing checklist

For a team starting from "we know we should have evals but don't":

- [ ] **Sprint 1, week 1.** 20-case golden set with rubric. Manual run. Pass-rate baseline established.
- [ ] **Sprint 1, week 2.** Cases in version control. Run-script committed. Results in shared destination.
- [ ] **Sprint 2.** LLM-as-judge harness. Calibration against the manual reviews. Judge agreement ≥ 85% gate before progressing.
- [ ] **Sprint 2.** Judge model pinned. Eval-runner config pinned in version control.
- [ ] **Sprint 3.** Fast / full suite split defined. Fast suite ≤ 10 minutes.
- [ ] **Sprint 3.** Eval gate wired into CI as required PR check. Threshold and override pattern defined.
- [ ] **Sprint 4.** Regression-case promotion process documented and enforced via PR-closing gate.
- [ ] **Sprint 4.** RAG-stage separation in eval (if RAG system). Retrieval recall scored separately from answer correctness.
- [ ] **Sprint 5.** Online sampling + thumbs/ratings pipeline. Judge-pass-rate SLI surfaced.
- [ ] **Sprint 6.** Quarterly recalibration on the calendar. Contamination audit on the calendar.
- [ ] **Sprint 7+.** Specialized suites (agent trajectory, adversarial / refusal) per workload need.

A team that completes this sequence has the eval practice that makes everything else in the engineering sibling repo possible.

---

## 16. References

- Anthropic, *Building evals*, *Evaluating prompts*, model-card eval methodology (2024–2026).
- OpenAI, evals framework documentation.
- Yann Dubois et al., *AlpacaFarm* and successor work on judge calibration.
- HELM (Stanford), MTEB, AgentBench — useful for model-level comparison and not for workload-specific eval.
- LangSmith, Braintrust, Phoenix, Helicone — eval platform vendors; engineering practice here is platform-agnostic.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](../../ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md) — system context for the worked example.
- Sibling repo: [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) — adversarial / red-team eval cases live in coordination with this repo's eval suites.
