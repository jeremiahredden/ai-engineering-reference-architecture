# Eval Anti-Patterns

> **Audience.** Engineers and tech leads whose team has *something called* an eval suite and is uncertain whether it is doing the job an eval suite should do. Anyone who has been told "we have evals" and looked at the evals and walked away unsettled but couldn't articulate why. Anyone who has had an eval suite pass while a production regression shipped past it. **Scope.** The seven eval anti-patterns I see most often, what they look like in the wild, what makes each one fail to function as an eval, and the structural fix for each. The closer doc for the eval-engineering folder; cross-referenced from every other eval doc as the negative-space counterpart to the positive-pattern docs. Pair with [eval-engineering-playbook.md](./eval-engineering-playbook.md) (the positive-pattern foundation) and every other doc in this folder (each anti-pattern points at the positive pattern that fixes it). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The hardest part of eval engineering is that broken evals look almost identical to working evals from the outside. A team that has an "eval suite" — a spreadsheet, a notebook, a CI job — that returns numbers and dashboards is doing eval engineering, in the same way a team that has a "test suite" of tests that always pass is doing test engineering. Whether the practice actually catches regressions, gates deployments, and protects users is invisible until something fails that the eval should have caught.

This document catalogs the seven failure patterns I see most often in 2026. Each one looks like eval from the outside. Each one fails to function as eval when it matters. The pattern is to recognize them by symptom, name them honestly, and fix the structural problem rather than papering over it with more eval activity.

The seven anti-patterns:

1. **Eval-as-vibe-check** — the team reads a handful of outputs and forms an opinion.
2. **Eval-as-spreadsheet** — manual scoring of static cases in a spreadsheet that nobody opens.
3. **Eval-as-judge-without-calibration** — LLM-as-judge whose calibration nobody has measured.
4. **Eval-as-leaderboard-on-public-benchmarks** — the team optimizes for HumanEval or MMLU instead of their workload.
5. **Eval-as-runs-only-when-someone-remembers** — eval exists but is not automated; runs sporadically; provides no gate.
6. **Eval-as-no-blocking-action** — eval runs and reports but cannot fail a deploy; the gate is decoration.
7. **Eval-as-passed-once-and-never-updated** — the suite was good in 2024; the workload has shifted; the suite hasn't.

The honest framing: most teams shipping production AI in 2026 have at least one of these patterns active. Naming them is the first step to fixing them. Each fix is structural — process change, infrastructure work, ownership clarification — not just "do more eval."

This document is opinionated about four things:

1. **An eval that cannot fail a deploy is not an eval.** It is observability. Both are valuable; only one is an eval.
2. **An eval with un-calibrated judges produces correlated-with-nothing pass-rates.** A pass-rate without calibration is a number, not a signal.
3. **An eval that drifts away from the production workload becomes anti-evidence: it gives confidence that the system is fine when production drift has rendered the eval irrelevant.**
4. **Naming the anti-pattern is the work.** "We have evals" is a statement that needs the question "do they function?" attached. The patterns here are how to answer.

Structure: (2)–(8) the seven anti-patterns, one section each, with symptom / why it fails / the structural fix; (9) worked Meridian example of recognizing and fixing multiple patterns; (10) the meta-anti-pattern (denying any of these apply); (11) findings; (12) adoption checklist; (13) references.

---

## 2. Anti-pattern 1: eval-as-vibe-check

### 2.1 What it looks like

The team has no formal eval suite. Quality is assessed by reading outputs:

- Engineer makes a prompt change.
- Engineer runs the agent on 5–10 example queries.
- Engineer reads the outputs and forms an impression.
- "Looks good" → ship. "Looks bad" → iterate.

There is no scoring rubric. There is no baseline. There is no persistence — the same 10 queries may not be re-run after the next change. There is no aggregation across changes — each evaluation is a moment-in-time judgment.

### 2.2 Why it fails

- **Inconsistent.** Different engineers form different impressions on the same outputs. The same engineer forms different impressions on different days.
- **Unaggregatable.** "It looks good" is not a number. Trend across changes is invisible.
- **Selection-biased.** Engineers tend to run queries that confirm their hypothesis. The queries the change *would* break are under-tested.
- **Not reproducible.** Six months later, "did this prompt change pass eval?" is unanswerable.
- **Tribal.** Quality assessment is in individual engineers' heads, not in artifacts.

### 2.3 The structural fix

Establish a minimum viable eval suite:

- 20 cases minimum, persisted in code/yaml.
- Per-case expected output or scoring rubric.
- Re-runnable on every change.
- Pass-rate tracked over time.

Per [eval-engineering-playbook.md §3](./eval-engineering-playbook.md), the 20-case golden set is the starting move. It does not need to be sophisticated; it needs to exist, be persisted, and be re-run.

### 2.4 The "we don't have time for evals" objection

The objection is wrong. The team is currently spending engineer-hours on vibe-checks; eval suite construction redirects that effort, doesn't add to it. The cost is a sprint of upfront work that pays back the first time it catches a regression that a vibe-check missed.

---

## 3. Anti-pattern 2: eval-as-spreadsheet

### 3.1 What it looks like

The team has an eval suite — in a spreadsheet. Cases are rows; expected outputs are a column; actual outputs are filled in manually after each change; a human scores each one as pass/fail in another column.

- No automation.
- Manual scoring takes hours per eval run.
- The spreadsheet is updated sporadically.
- "Did we re-run eval after that change?" is often answered "no, I just looked at a few cases."
- The spreadsheet is a snapshot of an eval *attempt*, not a continuous eval *practice*.

### 3.2 Why it fails

- **Manual-scoring throttle.** Because each run takes hours, runs happen rarely. The suite catches few regressions because it runs few times.
- **Stale.** The spreadsheet's "actual output" column reflects the last time someone ran the eval — not the current state.
- **No CI integration.** PRs merge without eval pass-check.
- **Brittle.** Adding cases means more manual labor; the suite stops growing past 50–100 cases.
- **Cross-team coordination tax.** Two engineers making concurrent changes both update the spreadsheet; the merge is painful.

### 3.3 The structural fix

Move the eval from spreadsheet to code:

- Cases in YAML/JSON or a code-defined test fixture.
- Auto-runner: invoke the agent, capture outputs, score against rubric (LLM judge or programmatic).
- CI integration: eval runs on every PR; results bubble to PR status.
- Persisted results: every run logged to a backing store with timestamps and version.

Per [eval-engineering-playbook.md §4](./eval-engineering-playbook.md), the eval is *code*, not a spreadsheet.

### 3.4 The "the spreadsheet is fine, we just need to update it more often" delusion

It isn't fine. The throttle is structural. Adding cases makes the throttle worse, not better. Update frequency stalls as the suite grows because manual scoring scales linearly with cases × runs. Automation is the only path.

---

## 4. Anti-pattern 3: eval-as-judge-without-calibration

### 4.1 What it looks like

The team has automated eval with an LLM judge. Cases run, the judge scores each output, a pass-rate is computed. Engineering looks at the pass-rate weekly.

Nobody has measured whether the judge actually agrees with human labels. Nobody knows the judge's variance — what would the pass-rate be if the same eval ran twice in a row? Nobody knows the judge's bias — does it prefer verbose answers? Conservative answers? Answers in a particular format?

The pass-rate is reported as if it were a measurement. It is not.

### 4.2 Why it fails

- **Pass-rate without calibration is uninterpretable.** "Pass-rate dropped from 91% to 88%" — is 3pp real or is it judge noise? Without calibration, the team can't tell.
- **Judge drift goes undetected.** If the judge's behavior changes (model version update, prompt tweak, scoring rubric reinterpretation), the pass-rate shifts without any underlying quality change. The team chases a phantom signal.
- **Judge bias compounds over time.** If the judge favors verbose answers, the team's prompts will gradually shift toward verbosity to score better. The eval optimizes for the judge's preference, not for users.
- **Cross-team eval results are not comparable.** Team A's judge and Team B's judge produce different pass-rates on the same outputs. Without calibration, comparison is meaningless.

### 4.3 The structural fix

Calibrate the judge:

- **Inter-rater agreement against humans.** Score 50–100 cases with humans + with the judge. Compute agreement (Cohen's kappa, raw agreement, or per-decision F1). Track over time.
- **Variance measurement.** Run the same eval twice on the same model+prompt+suite. Difference is the judge variance (the "flake floor" from [cicd-and-eval-gates/eval-gate-design.md §4.1](../cicd-and-eval-gates/eval-gate-design.md)).
- **Bias probes.** Construct cases that specifically test for verbose-vs-concise bias, format bias, refusal bias. Document the judge's biases.
- **Quarterly re-calibration.** When the judge model, prompt, or rubric changes, re-calibrate.

Per [llm-as-judge-patterns.md §6](./llm-as-judge-patterns.md), calibration is a first-class part of the practice.

### 4.4 The "we trust GPT-4 to judge" excuse

GPT-4 (or any frontier model) is *capable* of being a good judge, but is not *automatically* one. The capability is realized through calibration. Without calibration, the judge could be doing anything; the team has no way to tell.

---

## 5. Anti-pattern 4: eval-as-leaderboard-on-public-benchmarks

### 5.1 What it looks like

The team's "eval suite" is HumanEval, MMLU, MBPP, HELM, or similar public benchmarks. Engineers cite leaderboard scores when making model-selection decisions. New model versions are evaluated by running the public benchmarks.

The team's actual workload is not evaluated. Public benchmarks were never designed to predict performance on the team's specific tasks. The leaderboard score is a marketing artifact, not an eval artifact.

### 5.2 Why it fails

- **Workload-distribution mismatch.** HumanEval tests Python function completion; the team's workload is clinical query answering. The benchmark's signal does not transfer.
- **Contamination.** Many public benchmarks have been in training data for years. Frontier models can score high on them via memorization, not generalization. Pass-rates on contaminated benchmarks are inflated.
- **Optimization target divergence.** Vendors optimize for benchmark scores because they sell on benchmark scores. Capabilities that *don't* show up on the benchmark may be under-developed.
- **No production signal.** A model that improves on MMLU may regress on Care Coordinator queries. The leaderboard does not warn of this.

### 5.3 The structural fix

Build a workload-specific eval suite:

- Curate cases from your actual production workload (anonymized).
- Score against rubrics specific to your domain (clinical accuracy, format compliance, refusal-policy adherence).
- Use this suite for model-selection decisions. Public benchmarks are a sanity check, not a decision input.

Per [golden-set-design.md §2](./golden-set-design.md), the golden set is workload-derived. Public benchmarks are a starting reference, not a substitute.

### 5.4 The "we'll also run our own eval, but the leaderboard is faster" hybrid

The hybrid is fine if the leaderboard is treated as a sanity check, not a decision input. The failure mode is when leaderboard scores become the headline — the team's first metric, the model-selection driver, the discussion in standup — and the workload-specific eval becomes a secondary check.

If the leaderboard score and the workload eval disagree, trust the workload eval.

---

## 6. Anti-pattern 5: eval-as-runs-only-when-someone-remembers

### 6.1 What it looks like

The team has an eval suite. It is in code. It is well-designed. It runs when someone runs it.

- Not in CI.
- Not on a schedule.
- Run by the on-call engineer when something feels off.
- Run by the AI Platform team monthly when they update the dashboard.
- Run by the release author when they remember to.

Between runs, the system can degrade without anyone knowing. By the time someone runs it, the regression has been live for weeks.

### 6.2 Why it fails

- **Detection latency.** A regression shipped last Tuesday isn't caught until someone runs eval next Thursday. The window of harm is the gap.
- **Pre-release skip.** When the team is rushing a release, the manual eval run is the first thing dropped. The release that most needed eval got the least.
- **Tribal-knowledge dependency.** "Does anyone know if we ran eval before this last release?" is a question that has to be asked because nothing automated answers it.
- **Cross-team coordination.** Engineering doesn't know when eval ran last; product doesn't know whether to trust the dashboard.

### 6.3 The structural fix

Automate the eval run cadence:

- **Per-PR fast eval.** Runs automatically; results bubble to PR.
- **Per-release-candidate full eval.** Runs automatically on build.
- **Nightly full eval on main.** Catches drift since the last release.
- **Weekly extended eval.** Long-tail coverage.

Per [eval-gate-architecture.md](./eval-gate-architecture.md) and [cicd-and-eval-gates/pipeline-architecture-for-ai.md](../cicd-and-eval-gates/pipeline-architecture-for-ai.md), the cadence is automated. No one needs to remember.

### 6.4 The "we're a small team, we don't need automation" excuse

Small teams need automation *more*, not less. A small team can't afford the engineer-hour cost of manual eval runs. Automation moves the cost from per-run to one-time setup.

---

## 7. Anti-pattern 6: eval-as-no-blocking-action

### 7.1 What it looks like

The team has automated eval. It runs on every PR. It produces a report. The report is read.

The report does not block anything. The eval can fail; the PR can still merge. The eval can flag a 5pp pass-rate drop; the release can still ship. The eval is *informational*; the deploy gate is *advisory*.

### 7.2 Why it fails

- **Information without consequence is observability, not eval.** Both have value; only one prevents regressions from shipping.
- **The PR-merge incentive overrides the warning.** Engineers want their PR merged; the eval's "you might want to look at this" is naturally deprioritized vs the merge.
- **The eval becomes background noise.** When every PR has some eval warning, the warnings stop registering.
- **The first real regression to ship past the eval invalidates the entire suite.** Once the team knows "the eval said this would regress and it did, but we shipped it anyway," trust is permanently damaged.

### 7.3 The structural fix

Make the eval gate blocking:

- Per-PR fast eval: blocks PR merge on critical-case failure or pass-rate regression past threshold.
- Per-release full eval: blocks release promotion on regression.
- Override pattern with documented justification ([eval-gate-design.md §8](../cicd-and-eval-gates/eval-gate-design.md)).

The override pattern is crucial. A non-blocking gate fails because engineers route around it; a blocking gate with a documented override pattern handles legitimate exceptions while preserving the gate's force in normal cases.

### 7.4 The "we don't want to block engineers" framing

The framing is wrong. Engineers are not blocked; regressions are blocked. The override pattern lets engineers ship a documented exception when warranted. What the gate blocks is the *undocumented, unconsidered* regression — exactly what eval is there to prevent.

---

## 8. Anti-pattern 7: eval-as-passed-once-and-never-updated

### 8.1 What it looks like

The team's eval suite is a year old. It was designed when the system was different — a different model, a different prompt, a different feature set. The workload has shifted; the customer base has grown; new use cases have emerged.

The eval still runs. It still passes. The pass-rate is reassuringly stable.

Production behavior has shifted in ways the eval doesn't cover. New failure modes have emerged that the eval doesn't probe. The eval-passing system is producing user complaints; the team is confused why.

### 8.2 Why it fails

- **Production drift outpaces eval drift.** Production traffic mix shifts continuously; the eval suite is static. The eval becomes less and less representative.
- **New incidents don't get eval cases.** Every production incident is a discovered gap. If incidents don't generate eval cases, the suite stays blind to them.
- **Old cases become irrelevant.** Cases for features that were deprecated still run, padding the pass-rate; cases for new features that landed don't exist.
- **The eval becomes anti-evidence.** A passing eval that doesn't reflect production gives the team false confidence. The team would be better off knowing they have no eval than thinking they have one that works.

### 8.3 The structural fix

Treat the eval as a living artifact:

- **Quarterly suite review.** What cases are stale? What features added? What incidents happened that should generate cases?
- **Incident → case discipline.** Every production incident produces an eval case within the next sprint. [eval-engineering-playbook.md §6](./eval-engineering-playbook.md) makes this concrete.
- **Production-trace replay.** Sample production traffic into the eval suite periodically. Catches drift the team didn't anticipate.
- **Per-feature ownership.** Each feature has an eval owner; the owner is on the hook for keeping the feature's cases current.

### 8.4 The "the eval is working, don't fix what isn't broken" delusion

The eval may be broken; the symptom hasn't surfaced yet. The structural test: "Is the pass-rate today comparable to the pass-rate a year ago?" If yes, and the system has changed substantially in a year, the eval probably doesn't reflect the system anymore.

---

## 9. Worked Meridian example: recognizing and fixing multiple anti-patterns

The Care Coordinator team in early 2026 had several of these patterns simultaneously.

### 9.1 The starting state (Q1 2026)

- **Eval-as-vibe-check** for prompt changes: engineers ran 5–10 queries and read outputs.
- **Eval-as-spreadsheet** for monthly quality review: a 200-case spreadsheet that the AI Platform lead scored manually each month.
- **Eval-as-judge-without-calibration** for the small automated suite that existed: LLM judge was Claude 3.5 Sonnet at the time, never calibrated against human scoring.
- **Eval-as-runs-only-when-someone-remembers** for the automated suite: no CI integration; ran when someone kicked it off.
- **Eval-as-no-blocking-action** in the rare case it did run: results were posted to Slack; no deploy gate.

Five of the seven patterns. Outcome: a series of quality regressions reaching production that "the eval should have caught" but didn't because the eval wasn't structurally functional.

### 9.2 The remediation sequence

**Sprint 1: stop the bleeding.**
- Convert the spreadsheet to YAML cases.
- Build a CI runner that invokes the agent against the YAML cases.
- Wire the runner to GitHub Actions to run per-PR.
- Block PR merge on critical-case failure (start with 10 critical cases).

This addressed eval-as-vibe-check, eval-as-spreadsheet, eval-as-runs-only-when-someone-remembers (partial), and eval-as-no-blocking-action (partial).

**Sprint 2: calibrate the judge.**
- Pick 80 cases from the suite.
- AI Platform lead and one clinician score them by hand.
- Run judge against the same 80 cases.
- Compute Cohen's kappa: 0.61. Acceptable for production.
- Document the judge's biases (slight preference for verbose answers, slight under-refusal).
- Quarterly re-calibration scheduled.

This addressed eval-as-judge-without-calibration.

**Sprint 3: expand and own.**
- Convert all 200 spreadsheet cases to YAML.
- Designate per-feature owners.
- Build the production-trace replay pipeline (per [eval-of-agents.md §4.5](./eval-of-agents.md)).
- Add the incident-becomes-case discipline.

This addressed eval-as-passed-once-and-never-updated.

**Sprint 4: harden the gate.**
- Per-release-candidate full-eval gate.
- Nightly full eval on main.
- Override mechanism with documented justification.
- Gate health metrics published quarterly.

This closed eval-as-no-blocking-action and eval-as-runs-only-when-someone-remembers.

### 9.3 What changed in production

Quality incidents reaching production dropped from ~1.4/month to ~0.3/month over the following two quarters. The remediation cost ~2 engineer-quarters; payback was immediate.

### 9.4 What remained

Eval-as-leaderboard-on-public-benchmarks was never a Care Coordinator pattern — the team had been workload-focused from the start. The other six all needed structural remediation.

### 9.5 Findings closed

- **ARCH-CARE-099** (eval-as-vibe-check was the primary prompt-change quality gate).
- **ARCH-CARE-100** (eval-as-spreadsheet throttled run cadence).
- **ARCH-CARE-101** (LLM judge un-calibrated; pass-rates were correlated with judge bias not user quality).
- **ARCH-CARE-102** (eval ran only when triggered manually; weeks-long regression windows).
- **ARCH-CARE-103** (eval was advisory; no deploy gate; eval-flagged regressions shipped).
- **ARCH-CARE-104** (eval suite was static; production drift had rendered it less representative; "passing eval" gave false confidence).

---

## 10. The meta-anti-pattern: denying any of these apply

The strongest indicator that a team has at least one of these patterns is the response "none of these apply to us."

In practice, almost every team I've worked with — including ones with sophisticated AI-engineering practices — has at least one. The patterns are not exotic; they are default failure modes for a discipline that doesn't yet have a settled canon.

The honest stance:

- Walk through the seven patterns explicitly.
- For each, ask "does this describe any part of our practice?"
- Be willing to find at least one that does.
- The first remediation pays back fast; the team's eval practice improves measurably within a quarter.

A team that says "we're fine" without doing the walkthrough is almost certainly not fine. A team that does the walkthrough, finds one or two patterns, and remediates them is in much better shape than a team that pre-emptively denied any of them apply.

---

## 11. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| EVAL-AP-001 | Quality assessment by vibe-check; no persisted cases | High | Build minimum-viable 20-case suite per [eval-engineering-playbook.md §3](./eval-engineering-playbook.md) | Eval Eng + AI Platform |
| EVAL-AP-002 | Eval cases in spreadsheet; manual scoring per run | High | Convert to YAML / code fixtures; automate scoring per §3.3 | Eval Eng |
| EVAL-AP-003 | LLM judge un-calibrated; pass-rate interpretability unknown | High | Calibrate per [llm-as-judge-patterns.md §6](./llm-as-judge-patterns.md); quarterly re-anchor | Eval Eng |
| EVAL-AP-004 | Public benchmarks (HumanEval / MMLU) treated as primary eval signal | High | Build workload-specific suite per [golden-set-design.md](./golden-set-design.md); benchmarks are sanity check only | Eval Eng + AI Platform |
| EVAL-AP-005 | Eval runs only when triggered manually; no CI / scheduled cadence | High | Automate per §6.3 cadence; integrate with CI | Eval Eng + AI Platform |
| EVAL-AP-006 | Eval is advisory; not in deploy gate; regressions ship past flagged warnings | High | Make blocking with documented override pattern per [eval-gate-design.md §8](../cicd-and-eval-gates/eval-gate-design.md) | AI Platform + SRE |
| EVAL-AP-007 | Eval suite static; not updated with production drift or new incidents | High | Quarterly suite review; incident-to-case discipline per §8.3 | Eval Eng |
| EVAL-AP-008 | Judge bias not documented; prompt evolution may be optimizing for judge | Medium | Bias probes per §4.3; documented in judge config | Eval Eng |
| EVAL-AP-009 | Cross-team eval results not comparable due to differing judge configs | Medium | Shared judge config across teams or per-team calibration with bias documentation | Eval Eng + AI Platform |
| EVAL-AP-010 | Production-trace replay not used; suite drifts from production traffic mix | Medium | Production-trace replay per [eval-of-agents.md §4.5](./eval-of-agents.md) | Eval Eng + Observability |
| EVAL-AP-011 | Per-feature eval ownership undefined; suite drifts as features evolve | Medium | Designate per-feature owner; on-the-hook for keeping cases current | AI Platform + Eval Eng |
| EVAL-AP-012 | Stale eval cases not pruned; deprecated-feature cases pad pass-rate | Low | Quarterly review per §8.3; retire stale cases | Eval Eng |
| EVAL-AP-013 | "We don't have time for evals" used as rationale for vibe-check | Medium | Reframe: eval is redirecting effort, not adding it; cost is upfront, payback per regression caught | AI Platform Lead + Eval Eng |
| EVAL-AP-014 | "We trust the judge model" used as rationale for skipping calibration | Medium | Capability ≠ automatic correctness; calibration is the realization step | Eval Eng |
| EVAL-AP-015 | "Blocking gates slow engineers" used as rationale for advisory-only eval | Medium | Gate blocks regressions, not engineers; override pattern handles legitimate exceptions | AI Platform |
| EVAL-AP-016 | "None of these patterns apply" without explicit walkthrough | High | Run §10 walkthrough explicitly; find at least one pattern; remediate | AI Platform Lead + Eval Eng |
| EVAL-AP-017 | Eval results posted only to chat / dashboards; no formal record | Low | Persist results per [eval-gate-design.md §5.4](../cicd-and-eval-gates/eval-gate-design.md); audit trail | Eval Eng + Observability |
| EVAL-AP-018 | Manual eval runs prioritized last when releases are rushed; eval-most-needed releases get eval-least | High | Automated cadence per §6.3; manual is not the primary path | AI Platform |

---

## 12. Adoption checklist

- [ ] §10 walkthrough done; at least one pattern identified and acknowledged.
- [ ] Persistent eval cases (not vibe-check); minimum 20 cases.
- [ ] Cases in code/YAML, not spreadsheet.
- [ ] Automated runner; CI integration; no manual scoring.
- [ ] LLM judge calibrated against humans; calibration metric tracked; re-calibrated quarterly.
- [ ] Workload-specific suite is the primary signal; public benchmarks are sanity-check only.
- [ ] Automated cadence: per-PR fast, per-release-candidate full, nightly main, weekly extended.
- [ ] Eval gate is blocking with documented override pattern.
- [ ] Quarterly suite review; incident-to-case discipline; stale cases pruned.
- [ ] Per-feature eval ownership designated.
- [ ] Judge bias documented; bias-probe cases in suite.
- [ ] Production-trace replay pipeline; suite reflects production drift.
- [ ] Results persisted; audit trail.
- [ ] Override usage tracked and audited.
- [ ] Anti-pattern walkthrough re-run quarterly.

---

## 13. References

**Internal:**

- [eval-engineering-playbook.md](./eval-engineering-playbook.md) — the positive-pattern foundation.
- [golden-set-design.md](./golden-set-design.md) — the workload-specific case-design discipline.
- [llm-as-judge-patterns.md](./llm-as-judge-patterns.md) — judge calibration discipline.
- [regression-eval-suites.md](./regression-eval-suites.md) — incident-to-case practice.
- [online-eval-and-feedback.md](./online-eval-and-feedback.md) — production signals.
- [eval-gate-architecture.md](./eval-gate-architecture.md) — gate placement.
- [eval-of-rag.md](./eval-of-rag.md) — RAG-specific positive patterns.
- [eval-of-agents.md](./eval-of-agents.md) — agent-specific positive patterns.
- [cicd-and-eval-gates/eval-gate-design.md](../cicd-and-eval-gates/eval-gate-design.md) — the gating pattern this complements.
- [cicd-and-eval-gates/pipeline-architecture-for-ai.md](../cicd-and-eval-gates/pipeline-architecture-for-ai.md) — pipeline that hosts the gate.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — production-side drift signal.
- [prompt-engineering/prompt-anti-patterns.md](../prompt-engineering/prompt-anti-patterns.md) — companion anti-pattern catalog.
- [agent-engineering/agent-anti-patterns.md](../agent-engineering/agent-anti-patterns.md) — companion anti-pattern catalog.

**Cross-repo (architecture sibling):**

- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
