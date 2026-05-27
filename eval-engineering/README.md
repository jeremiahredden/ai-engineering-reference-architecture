# Eval Engineering

## What this folder is

The engineering discipline of evaluating AI systems — building eval suites, running them on a schedule, gating CI on their results, calibrating LLM judges, and treating the eval surface as a first-class engineering artifact owned by the same team that owns the feature. The material here is what I put in front of a team when the question is: *we know we are supposed to "have evals" but what does that actually mean in practice, what do we build first, and how do we keep it from becoming a wiki page that nobody updates?*

## The organizing principle

Evals are the foundational engineering discipline for AI systems. Without an eval suite a team cannot safely deploy, roll back, fine-tune, change models, refactor prompts, or know whether a "fix" actually fixed anything. Worse — without evals, *the team will not know that they cannot do these things*, because nothing will break loudly until it is too late.

But evals are also the discipline that teams most often defer, because the upfront investment looks heavy. So the patterns here are explicitly designed for *incremental adoption*: a 20-case golden set you can ship in a sprint, an LLM-as-judge harness the sprint after, an eval gate in CI the sprint after that. Each step is independently useful; together they compound into the engineering practice that makes everything else in this repo possible.

The folder is opinionated about three things specifically. First, that a 20-case golden set with binary pass/fail criteria and a manual review process is worth more than a 2,000-case eval suite that nobody trusts the scoring on. Second, that LLM-as-judge is the only practical pattern for many quality dimensions, but it must be calibrated against human judgment before its outputs are trusted as a gate. Third, that evals only matter if they *gate something* — if the eval fails and a release ships anyway, the eval is a dashboard, not a discipline.

## Planned documents

- **[eval-engineering-playbook.md](./eval-engineering-playbook.md)** — Soup-to-nuts playbook on building an eval practice from zero. The 20-case golden set as the starting move, eval-driven development as the workflow, LLM-as-judge calibration discipline, regression suites, online-eval signals (thumbs-up/down, retry rate, escalation rate), eval-gate placement in CI, the integration with sibling observability content, sprint sequencing from zero to GA, six anti-patterns, the worked Meridian Care Coordinator example, and sprint-assigned EVAL- findings.
- **[golden-set-design.md](./golden-set-design.md)** — How to design a golden set that pays back: case selection (representative, adversarial, rare-but-critical), expected-output formats, scoring rubrics, ownership and refresh discipline, and the anti-pattern of golden sets that grew organically and now nobody knows what coverage they actually provide.
- **[llm-as-judge-patterns.md](./llm-as-judge-patterns.md)** — The patterns for using an LLM as the evaluator: pairwise comparison, scalar scoring, rubric-based, structured-criteria. Calibration against human labels, judge-model selection (typically a more capable model than the system under test), judge-prompt versioning, and the failure modes (judge agrees with itself too much, judge biased toward verbose answers, judge drifts with model version changes).
- **[regression-eval-suites.md](./regression-eval-suites.md)** — Building regression suites from real bugs and real incidents — every fixed quality issue becomes a permanent eval case so it cannot regress silently. Suite organization, subset selection for fast CI feedback, and the discipline of running the full suite nightly and on release candidates.
- **[online-eval-and-feedback.md](./online-eval-and-feedback.md)** — The production-side eval signals: explicit feedback (thumbs, ratings, free-text), implicit signals (retry, edit, abandonment, escalation to human), and the design that lets online signals feed back into the golden set without becoming a noisy mess. Integration with the engineering sibling's `observability-and-telemetry/` folder.
- **[eval-gate-architecture.md](./eval-gate-architecture.md)** — Where eval gates sit in the CI / CD pipeline, the threshold-setting discipline (block on pass-rate drop > X%), the per-PR-fast-suite vs nightly-full-suite split, and the override pattern for cases where a regression is intentional. Aligned with sibling `cicd-and-eval-gates/`.
- **[eval-of-agents.md](./eval-of-agents.md)** — Evaluating multi-step agents — much harder than evaluating single LLM calls. Trajectory eval, step-level eval, outcome-only eval, tool-call accuracy eval, and the cost-aware sampling strategies that keep the eval suite affordable when each case costs dollars to run.
- **[eval-of-rag.md](./eval-of-rag.md)** — RAG-specific eval dimensions: retrieval recall, retrieval precision, citation accuracy, faithfulness (answer is grounded in retrieved content), and the eval harness that separates retrieval failures from generation failures so the diagnostic is actionable.
- **[eval-anti-patterns.md](./eval-anti-patterns.md)** — The seven patterns I see teams adopt that look like evals but do not function as evals: eval-as-vibe-check, eval-as-spreadsheet, eval-as-judge-without-calibration, eval-as-leaderboard-on-benchmarks-not-your-workload, eval-as-runs-only-when-someone-remembers, eval-as-no-blocking-action, and eval-as-passed-once-and-never-updated.

## How to use this section

**If your team does not have evals yet**, read [eval-engineering-playbook.md](./eval-engineering-playbook.md) end-to-end. It is the on-ramp; the rest of the folder is the deeper practice once the on-ramp is built.

**If you have evals but they do not gate anything**, `eval-gate-architecture.md` is the next move. An eval that does not gate is a dashboard; a dashboard does not prevent regressions.

**If you have evals that nobody trusts the scoring on**, the problem is almost always in `llm-as-judge-patterns.md` — uncalibrated judges, drifted judges, or judges scoring the wrong dimension. The fix is calibration discipline, not a different judge model.

## What this section is not

- **A benchmark guide.** MMLU, MTEB, HELM, AgentBench, and the academic benchmark canon are useful for model comparison and almost useless as a quality signal for your specific workload. This folder is about workload-specific evals.
- **A research methodology guide.** Open questions in eval research (judge drift, contamination bounding, calibrated hallucination rate estimation) are out of scope; the patterns here are the working-engineer's practice that handles the cases the research is still debating.
