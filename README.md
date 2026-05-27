# AI Engineering Reference Architecture

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Maintained](https://img.shields.io/badge/Maintained-yes-green.svg)
![Sibling repo: ai-architecture-reference-architecture](https://img.shields.io/badge/sibling-ai--architecture--reference--architecture-orange.svg)
![Sibling repo: ai-security-reference-architecture](https://img.shields.io/badge/sibling-ai--security--reference--architecture-orange.svg)
![Sibling repo: appsec-reference-architecture](https://img.shields.io/badge/sibling-appsec--reference--architecture-orange.svg)
![Sibling repo: cloud-security-reference-architecture](https://img.shields.io/badge/sibling-cloud--security--reference--architecture-orange.svg)

### A practitioner's toolkit for shipping AI systems — eval engineering, prompt engineering, RAG engineering, agent engineering, model lifecycle, observability, reliability, CI/CD with eval gates, and AI FinOps

**Jeremiah Redden** | Senior AI/AppSec Security Architect | CISSP | [github.com/jeremiahredden](https://github.com/jeremiahredden)

---

This repository is the *shipping and operating* side of the AI build problem. Its closest sibling, [**ai-architecture-reference-architecture**](https://github.com/jeremiahredden/ai-architecture-reference-architecture), is the *design* side — pattern catalogues, model strategy, data architecture, integration shapes, and reference systems. The third closely-related sibling, [**ai-security-reference-architecture**](https://github.com/jeremiahredden/ai-security-reference-architecture), covers how to attack and defend the systems described here. All three are designed to be read together. Where the architecture sibling says *"this is the pattern,"* this repo says *"and here is how you eval it, instrument it, ship it, roll it back, and operate it on a budget."*

The split is deliberate. AI engineering is its own discipline now — distinct from ML engineering (which is centered on training pipelines and model performance), distinct from data engineering (which is centered on pipelines and warehouses), and distinct from software engineering (which assumes deterministic functions and exhaustive tests). AI engineering is the practice of shipping software systems whose primary components are nondeterministic, expensive per-call, drift over time, and fail in ways that conventional testing does not catch. The disciplines that matter most — evals, prompt-as-code, agent loops, RAG observability, eval gates in CI, model rollbacks, FinOps for tokens — do not have a settled canon yet. This repo is one practitioner's attempt at one.

A second motivation: most public AI engineering material today is either vendor-driven ("here is how to use our SDK") or framework-driven ("here is how to use LangChain"). Neither helps a team that has already shipped one AI feature, has a real on-call rotation, has a real cost line on the finance dashboard, and is now asking *"how do we run this responsibly at scale?"* The patterns here are calibrated to that team — opinionated about which engineering disciplines pay back, explicit about the failure modes that show up only in production, and biased toward instrumentation and rollback over heroic recovery.

---

## Table of Contents

- **[eval-engineering/](./eval-engineering/)** — The engineering discipline of evals. Offline eval suites, online evals, LLM-as-judge with calibration, golden sets, regression suites, eval-driven development workflows, and the eval-gate patterns that let CI fail a PR for a quality regression the same way it fails for a unit test. The first deep document, [eval-engineering-playbook.md](./eval-engineering/eval-engineering-playbook.md), is landed.
- **[prompt-engineering/](./prompt-engineering/)** — Prompts as code: versioning, A/B testing, structured outputs (JSON Schema, tool calls, constrained decoding), prompt libraries and reusable components, prompt-as-API discipline, system-prompt vs few-shot vs retrieval composition, and the workflow that turns "the prompt is a string in someone's notebook" into "the prompt is a versioned artifact with an owner and a test suite."
- **[rag-engineering/](./rag-engineering/)** — Ingestion pipelines, document-aware chunking, embedding strategy (model selection, dimensions, batching, drift), retrieval at scale (BM25, vector, hybrid, filtered, multi-stage), reranking, query rewriting, retrieval observability (what was retrieved, what was used, what was ignored), and the RAG-specific failure modes — empty retrievals, stale retrievals, off-topic retrievals, and the eval coverage that catches each.
- **[agent-engineering/](./agent-engineering/)** — The engineering practice of shipping agents. Agent loop design, tool-call architecture, planning vs reactive loops, memory (short-term, long-term, episodic, semantic), error recovery, partial failure handling, multi-agent coordination, eval strategies specific to agents, and the operational disciplines that keep an agent useful instead of expensive. The first deep document, [agent-engineering-playbook.md](./agent-engineering/agent-engineering-playbook.md), is landed.
- **[model-lifecycle/](./model-lifecycle/)** — Model registry, model promotion (dev → staging → prod), fine-tuning operations, distillation, A/B model rollout, canary and shadow patterns, rollback procedures, model-version-as-dependency discipline, runtime platform integration (vLLM, TGI, Triton, hosted endpoints), and the model-deprecation pattern that lets a team migrate off a sunset model without breaking production.
- **[data-engineering-for-ai/](./data-engineering-for-ai/)** — Labeling and annotation workflows, dataset versioning, synthetic data generation, data quality for AI (label noise, distribution drift, contamination), training/eval split discipline, retrieval-corpus engineering, and the data contracts that prevent a silent retrieval-source change from silently breaking the model behavior downstream.
- **[observability-and-telemetry/](./observability-and-telemetry/)** — Trace and span design for AI systems, LLM call instrumentation (tokens, latency, cost, model, prompt version), retrieval instrumentation, agent-step instrumentation, quality drift detection, cost dashboards, alerting patterns, and the OpenTelemetry-for-LLMs conventions that make AI systems debuggable in production the way conventional systems already are.
- **[reliability-engineering/](./reliability-engineering/)** — Timeout strategy, retry strategy (and when retries are wrong for non-idempotent agent steps), fallback patterns (model fallback, retrieval fallback, degraded responses), circuit breakers, fault budgets for AI systems, capacity planning for spiky workloads, the multi-provider failover pattern, and the design discipline that distinguishes "the model is slow" from "the model is broken" in production.
- **[cicd-and-eval-gates/](./cicd-and-eval-gates/)** — Pipeline architecture for AI systems, the eval-gate-as-quality-gate pattern, canary rollouts, shadow traffic, prompt-version pinning in releases, model-version pinning, dataset-version pinning, branch-protection rules that prevent merge without eval pass, and the CI workflow that gives an AI PR the same rigor a database-migration PR already gets.
- **[cost-and-finops/](./cost-and-finops/)** — Token-level cost accounting, per-feature and per-tenant chargeback, model-tier routing for cost, caching tiers for cost, batch vs realtime cost optimization, cost-aware rate limiting, the cost-budget-as-circuit-breaker pattern, and the FinOps-for-AI discipline that prevents a successful launch from becoming a budget incident.

---

## How to Use This Repo

**If you are a staff engineer or tech lead** shipping your team's first production AI feature, [eval-engineering/](./eval-engineering/) is non-negotiable and should be read first. Evals are the engineering practice that makes everything else in this repo possible — without an eval suite you cannot safely deploy, roll back, fine-tune, change models, or refactor prompts. After evals, [observability-and-telemetry/](./observability-and-telemetry/) is the second-most-leveraged investment.

**If you are an AI platform engineer** building the platform other product teams will build on, [model-lifecycle/](./model-lifecycle/), [cicd-and-eval-gates/](./cicd-and-eval-gates/), and [observability-and-telemetry/](./observability-and-telemetry/) are the three platform-shaped investments. The platform's job is to make the boring-but-essential parts uniform so product teams can focus on the actual feature.

**If you are an ML or AI engineer** shipping retrieval systems, [rag-engineering/](./rag-engineering/) is the destination. RAG looks simple in the architecture diagram and breaks in twenty places in production; this section documents the twenty places.

**If you are shipping agents**, [agent-engineering/](./agent-engineering/) is where the on-call-grade engineering patterns live. Read it alongside the sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture)'s `agent-security/` folder — agents fail both in correctness and in safety, and the engineering practice has to address both.

**If you are an SRE or DevOps engineer** integrating AI systems into the broader reliability practice, [reliability-engineering/](./reliability-engineering/) and [observability-and-telemetry/](./observability-and-telemetry/) translate AI-specific failure modes into the SRE vocabulary already in use (SLOs, error budgets, runbooks, paging policy).

**If you are a finance or engineering leader** trying to make AI cost predictable, [cost-and-finops/](./cost-and-finops/) is the section. AI cost is not a procurement problem — it is an engineering discipline. The patterns there are what makes a launch not turn into a budget incident.

**If you are a hiring manager** evaluating my work, the worked playbooks in [eval-engineering/eval-engineering-playbook.md](./eval-engineering/eval-engineering-playbook.md) and [agent-engineering/agent-engineering-playbook.md](./agent-engineering/agent-engineering-playbook.md) are closer to my deliverables than a resume will ever be. Read those first.

---

## Philosophy

Four principles drive every piece of content in this repository. They are the same four that govern the sibling repos, restated for AI engineering.

**1. AI engineering should make product teams faster, not slower.** An engineering practice that requires a six-week eval-suite build before the first feature ships is not engineering — it is delay disguised as rigor. The eval, observability, and CI patterns in this repo are designed for incremental adoption: ship a 20-case golden set in a sprint, an LLM-as-judge harness the sprint after, an eval gate in CI the sprint after that. Each is independently useful; together they compound. I optimize for engineering investments that pay back inside a quarter, not ones that require a quarter of upfront work before the first benefit shows up.

**2. The right engineering investment is the one that ships this sprint.** The single biggest failure mode in AI engineering in 2026 is over-investment in tooling that the system doesn't yet need — building a full LLMOps platform when there are two LLM calls in production, building a fine-tuning pipeline before there is a clear capability gap, building a multi-provider routing layer before there is a single-provider quality problem. Complexity in AI infrastructure compounds nonlinearly: every additional component is more cost, more failure modes, and more maintenance. Every pattern here names the simpler version that should be tried first and the signals that justify the more complex one.

**3. Every AI engineering finding needs a fix an engineer can land in the current sprint.** A quality dashboard that says "model performance is degrading" has failed. A quality dashboard that says "judge-pass-rate on the medication-dosing golden subset dropped from 94% to 81% on 2026-05-22, root cause appears to be the upstream guidelines corpus refresh on 2026-05-21 which changed the dosing-table HTML structure, owner: ai-platform-eng, tracked as EVAL-046, by sprint 47" has succeeded. Every template, runbook, and eval pattern in this repo is written to that standard.

**4. An AI engineering practice that only exists in a wiki doesn't exist. Show your work.** Process documents, framework comparisons, and "AI maturity model" diagrams are evidence of intent, not of engineering. Where the patterns here can be turned into runnable artifacts — an eval suite that fails CI on regression, an observability stack that produces real traces, a rollback runbook that has been exercised, a cost dashboard that closes the loop on a budget breach — they are. Every engineering claim should be backed by an artifact a reviewer can execute or a metric they can verify.

---

## What this repo is not

- **A vendor walkthrough.** Where specific tools are named (Anthropic, OpenAI, Cohere, LangSmith, Braintrust, Phoenix, Helicone, LangChain, LlamaIndex, Vercel AI SDK, vLLM, TGI), it is because the pattern is easier to explain concretely. Substitute the vendor of your choice; the discipline is what matters.
- **A "build your own LLMOps platform" project.** The discipline matters more than the platform. A team using LangSmith for traces and Braintrust for evals and GitHub Actions for CI is doing AI engineering well; a team that built a custom platform around all three and is now maintaining it instead of shipping features is not. The patterns here are platform-agnostic where it is practical.
- **A research methodology guide.** Eval methodology has deep open research questions (how to design LLM judges that don't drift, how to bound contamination, how to estimate hallucination rates with calibration). Where they intersect with engineering practice they are addressed; the rest is out of scope.
- **AI architecture or AI security.** The design side lives in [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture); the attack-and-defend side lives in [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture). Use all three together; the boundaries are deliberate.

---

## License & Attribution

All content in this repository is released under the MIT License unless otherwise noted. Templates, playbooks, eval patterns, and runbooks may be used, modified, and adapted freely — including in commercial engagements and client deliverables. Attribution is appreciated but not required.

If you find something useful, I would like to hear about it. Open an issue, reach out on LinkedIn, or send a pull request with your improvements.
