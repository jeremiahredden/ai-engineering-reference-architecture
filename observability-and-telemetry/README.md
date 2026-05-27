# Observability and Telemetry

## What this folder is

The engineering practice of making AI systems debuggable in production — traces, spans, structured logs, metrics, dashboards, and alerting designed around the shapes AI systems actually take (LLM calls, retrieval calls, agent loops, tool calls). The material here is what I put in front of a team when the question is: *the chat feature is "feeling slower" and "getting worse," support is sending screenshots, and the only logging we have is the request line and HTTP status — how do we get a real signal on what is happening?*

## The organizing principle

Observability for AI systems shares about 60% of its discipline with general distributed-systems observability (OpenTelemetry, trace / span / log / metric, RED and USE methods, percentile-based SLOs, etc.) and adds about 40% that is genuinely new (per-call token accounting, prompt-version-as-trace-attribute, model-version-as-trace-attribute, retrieved-doc-IDs-as-span-attribute, judge-pass-rate-as-quality-SLI, cost-per-feature-per-tenant). The new 40% is what makes AI debugging tractable. Without it, an AI quality regression is investigated the way 2010-era latency regressions were investigated — by guessing and shipping experiments.

So the patterns here are arranged around *attributes-first* observability: instrument every LLM call, every retrieval call, every agent step, every tool call with the attributes that make later debugging possible (prompt version, model version, model parameters, input/output tokens, cost, latency-to-first-token, latency-total, retrieved doc IDs, judge score if scored). And then arrange those signals into the SLI / SLO / alert hierarchy that integrates AI quality with the rest of the reliability practice.

The folder is opinionated about three things specifically. First, that *traces are the primary debugging surface for AI systems* — single-call logs are too narrow and aggregate metrics are too coarse; the trace of a single failing interaction is what makes the bug actionable. Second, that *cost is a first-class telemetry signal*, not a billing dashboard line — cost-per-call attributed to feature / tenant / model / prompt-version is the early-warning system for regressions and abuse. Third, that *quality SLOs* (judge-pass-rate on the production stream, hallucination rate, citation accuracy) belong in the same alerting hierarchy as latency and availability SLOs.

## Planned documents

- **trace-and-span-design.md** — Trace and span structure for AI workloads: one trace per top-level user request, one span per LLM call / retrieval call / tool call / agent step, the attributes to capture (prompt version, model version, token counts, cost, latency components, retrieved doc IDs, judge scores), and the OpenTelemetry-for-LLMs conventions emerging in 2025-2026.
- **llm-call-instrumentation.md** — The wrapper pattern that gives every LLM call uniform instrumentation: provider, model, version, parameters, input tokens (cached and uncached), output tokens, cost (computed at request time, not after the fact), latency-to-first-token, total latency, finish reason, tool-call presence. The integration with the architecture sibling's `ai-gateway-pattern`.
- **retrieval-instrumentation.md** — Retrieval span attributes: query, query-rewrite if any, retrieved doc IDs, retrieval scores, reranker scores, doc-corpus version, embedding model version, retrieval latency, post-filter count. The per-call retrieval observability that makes "why did the model say that" answerable.
- **agent-step-instrumentation.md** — Per-loop-turn spans, per-tool-call spans, the budgets-as-attributes pattern (turn-budget remaining, cost-budget remaining, time-budget remaining), and the trace-as-debugging-surface discipline for agents.
- **quality-drift-detection.md** — Quality-as-SLI: judge-pass-rate, retry-rate, escalation-rate, citation-accuracy, format-validation-pass-rate. The drift-detection patterns and the alert design that distinguishes "model is broken" from "the world changed."
- **cost-dashboards.md** — Cost attribution at the granularity that supports action: cost-per-feature, cost-per-tenant, cost-per-user, cost-per-model. The cost-spike-alert design that catches a runaway feature before it shows up in the monthly invoice.
- **vendor-tool-integration.md** — Integration patterns for LangSmith, Braintrust, Phoenix, Helicone, Datadog LLM Observability, and the OpenTelemetry path. The build-vs-buy decision tree and the avoid-vendor-lock-in patterns (instrument once, export to multiple destinations).
- **alerting-and-paging-design.md** — Which AI signals page (cost spike > X, judge-pass-rate drop > Y, retrieval-empty-rate > Z, agent-loop-runaway > W), which alert without paging, which go to dashboard only, and the runbook pattern that the alert points at.
- **debugging-from-traces.md** — The diagnostic playbook for AI debugging from traces: read the trace top-down, separate retrieval failures from generation failures, separate prompt regressions from model regressions, separate tool failures from agent-misuse-of-tools. The repeatable diagnostic discipline.

## How to use this section

**If your AI system has no AI-specific observability**, `llm-call-instrumentation.md` is the first move. The wrapper pattern can be deployed in a single sprint and starts producing actionable signal immediately.

**If you have logs but cannot answer "why did the model say that"**, `trace-and-span-design.md` and `retrieval-instrumentation.md` together describe the upgrade. Traces beat logs for AI debugging; retrieval attributes beat opaque embeddings for explainability.

**If your AI cost is growing faster than usage**, `cost-dashboards.md` is the diagnostic. Most teams discover the cause is a single feature or a single user-cohort whose per-call cost is 10–100x the median.

## What this section is not

- **A general observability primer.** Distributed-systems observability (OpenTelemetry, RED, USE, SRE practices) is well-covered elsewhere. This folder is about the AI-specific overlays.
- **A vendor recommendation.** Where LLM observability vendors are named, they are illustrative; the discipline is what matters.
