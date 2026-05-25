# Debugging from Traces

> **Audience.** Engineers debugging AI features in production. Anyone who has been paged for "the AI feature did something weird" and needs a structured approach beyond re-running. **Scope.** The *engineering* practice of reading AI traces to diagnose failures. Top-down workflow, span-by-span analysis, common patterns. Pair with [trace-and-span-design.md](./trace-and-span-design.md), [rag-failure-modes-and-debugging.md](../rag-engineering/rag-failure-modes-and-debugging.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The trace is the primary debugging surface for AI systems. Without trace discipline, debugging is "rerun the question and see if it happens again" — which is unreliable for nondeterministic systems and impossible for past incidents whose state has since changed. With trace discipline, the captured trace is the evidence; investigations read the trace and arrive at a diagnosis.

The [rag-failure-modes-and-debugging.md](../rag-engineering/rag-failure-modes-and-debugging.md) document covers RAG-specific failure modes. This document is the broader cross-cutting practice — how to *read* traces for any AI debugging, the patterns common across system types, the workflow that makes investigations fast.

This document is opinionated about three things:

1. **The trace is the evidence; re-running is a secondary tool.** Investigations start with the captured trace; new experiments come later if at all.
2. **The workflow is top-down.** Read the outer span first; descend only into stages that look suspicious. Don't dive into LLM-call attributes before checking the agent loop's outcome.
3. **Pattern recognition compounds.** As the team accumulates investigation experience, common patterns get faster to identify. The patterns are documented.

Structure: (2) the trace-reading workflow; (3) the top-down read; (4) per-span-type debugging patterns; (5) common cross-cutting patterns; (6) when re-running is appropriate; (7) the diagnosis-to-fix loop; (8) integration with the failure-mode catalog; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The trace-reading workflow

The structured workflow for reading a trace.

### 2.1 The workflow phases

```
Intake
    │ Identify the affected interaction; pull the trace
    ▼
Top-down read
    │ Read the outermost span; descend on suspicion
    ▼
Hypothesis formation
    │ What stage is the likely cause?
    ▼
Deep-dive
    │ Inspect the suspicious span's attributes
    ▼
Diagnosis
    │ Root cause identified
    ▼
Fix + regression case
```

The phases are sequential; each phase produces information for the next.

### 2.2 The intake

Before reading the trace:

- **Symptom.** What does the user (or alert) report?
- **Interaction ID.** The trace ID for the affected interaction.
- **Timing.** When did it happen? (Confirms the trace retention covers the period.)
- **Recurrence.** Is this happening to many interactions, or one?

The intake frames the investigation.

### 2.3 The trace pull

For platforms with multiple trace destinations:

- Pull from the primary destination (Datadog, LangSmith, etc.).
- If the trace is missing or incomplete: check secondary destinations.
- If the trace is sampled out (per [trace-and-span-design.md](./trace-and-span-design.md) section 5): the team may need to wait for the next occurrence or rely on tail-based augmentation.

The trace is the evidence; if it's not available, investigation is limited.

### 2.4 The workflow latency target

For Sev-1 / Sev-2 incidents: from page to root cause within 1 hour.

For lower-severity: within a business day.

The workflow's pacing supports these targets.

### 2.5 The workflow documentation

Each common failure class has a documented workflow (per [rag-failure-modes-and-debugging.md](../rag-engineering/rag-failure-modes-and-debugging.md) sections 3-6; per the broader runbook practice). The workflow accelerates investigation; new on-call engineers learn from it.

---

## 3. The top-down read

The systematic read pattern.

### 3.1 The outermost span

Start with the top-level interaction or workflow span:

- What's the outcome attribute? (success / escalated / terminated_budget / error)
- What's the total latency? (Within SLO?)
- What's the total cost? (Within budget?)
- How many turns / sub-spans? (Within expected range?)

The outermost span tells you whether things terminated cleanly or got stuck.

### 3.2 The first-level descendants

What's inside the outermost span?

- For agents (per [agent-step-instrumentation.md](./agent-step-instrumentation.md)): per-turn spans.
- For workflows: per-stage spans.
- For chains: each stage's span.

Are they in the expected order and count? Any unusual patterns (extra retries, missing stages)?

### 3.3 The pattern recognition at this level

- Normal: expected number of turns; clean termination.
- Loop-runaway: many more turns than expected.
- Early-termination: fewer turns than expected (budget breach, error).
- Stuck: long latency on one turn.

Each pattern suggests where to descend.

### 3.4 The selective descent

Don't read everything. Descend into the spans that look suspicious:

- If turn 5 had a tool call that returned an error, descend into that tool call.
- If LLM call span had unusual finish_reason, descend into it.
- If retrieval returned empty, descend into the retrieval spans.

Selective descent keeps investigation focused.

### 3.5 The pattern of zoom

```
Outer interaction
    ▼ (suspicion)
Workflow / agent turns
    ▼ (suspicion)
Specific LLM call or tool call or retrieval
    ▼ (deep-dive)
Span attributes (prompt version, retrieved docs, scores, etc.)
```

Each level zooms in; the team reads the level where the cause is.

### 3.6 The visualization

Most trace UIs (Datadog, LangSmith, Phoenix, etc.) show traces as expandable trees. The team uses the UI to navigate:

- Collapsed view: see the structure.
- Expanded selectively: dive into the spans of interest.

The UI choice affects investigation speed. Per [vendor-tool-integration.md](./vendor-tool-integration.md), teams often have multiple destinations; the UI used depends on the investigation type.

---

## 4. Per-span-type debugging patterns

The specific patterns for each common span type.

### 4.1 The LLM-call span

Per [llm-call-instrumentation.md](./llm-call-instrumentation.md), the LLM call span includes:

- Model + version + prompt version.
- Token counts (input cached / uncached / output).
- Latency components (TTFT, total).
- Finish reason.
- Cost.

Common patterns to check:

- **Wrong model version.** Alias resolved to a different version.
- **Wrong prompt version.** Released with the wrong pin.
- **Excessive output tokens.** Model produced more than expected; cost spike.
- **`length` finish reason.** Hit max-tokens; response was truncated.
- **`content_filter` finish reason.** Safety filter activated.
- **Cache hit rate dropped.** Prompt structure changed; uncached tokens up.

Each pattern suggests a specific cause.

### 4.2 The retrieval span

Per [retrieval-instrumentation.md](./retrieval-instrumentation.md) and [rag-failure-modes-and-debugging.md](../rag-engineering/rag-failure-modes-and-debugging.md):

- Query (or query hash).
- Returned doc IDs.
- Per-retriever sub-spans (BM25, vector, rerank).
- Scope verification result.

Common patterns:

- **Empty retrieval.** Filter too restrictive; corpus missing content.
- **Missed expected source.** Source not in top-K; recall issue.
- **Wrong tenant scope.** Sev-1 isolation event.
- **Rerank dropped a good chunk.** Rerank-score issue.

### 4.3 The tool-call span

Per [agent-step-instrumentation.md](./agent-step-instrumentation.md):

- Tool name + version.
- Authorization decision (allow / deny).
- Arguments hash.
- Success / failure.
- Side-effect linkage (proposal_id, draft_id).

Common patterns:

- **Authorization denied.** Caller lacks the role / scope; expected refusal or unexpected attack.
- **Tool failure.** Backend issue; need to investigate the tool.
- **Wrong tool selected.** Agent's classification was wrong.
- **Unexpected side effect.** HITL contract violation.

### 4.4 The agent-turn span

- Turn number; budget remaining (turn, cost, time, tool-call).
- Decision (tool_call / final_answer / escalate / terminate_budget).
- Per-worker sub-spans (for supervisor / worker).

Common patterns:

- **Budget exhausted.** Loop ran too long; check turn-by-turn to identify why.
- **Stuck in loop.** Repeated tool calls without progress.
- **Wrong worker dispatched.** Supervisor's classification was wrong.

### 4.5 The workflow stage span

- Stage name.
- Input and output handles.
- Stage-specific attributes.

Common patterns:

- **Stage skipped.** Conditional logic excluded the stage; verify the condition.
- **Stage error.** Stage failed; check error attributes.
- **Stage timeout.** Took longer than budget; check stage's own breakdown.

### 4.6 The cross-span correlation

Some debugging needs cross-span correlation:

- The LLM call's cost is high because the retrieval's chunk count is high.
- The agent's loop is stuck because the tool's response is malformed.

The team reads multiple spans and connects the dots; the UI's parent-child navigation supports this.

---

## 5. Common cross-cutting patterns

Patterns that appear across span types and feature types.

### 5.1 The "right answer to wrong question"

The model answered correctly given what it understood, but it understood the wrong question.

Signs:
- The query was ambiguous.
- The query was a conversational follow-up that the rewriter didn't capture correctly.
- The classifier dispatched to the wrong worker.

Investigation: read the query, the classifier's decision, the rewrite (if any), the model's response. Identify the misalignment.

Remediation: query rewriting tuning; classifier prompt; per [query-rewriting.md](../rag-engineering/query-rewriting.md).

### 5.2 The "model ignored retrieved context"

Retrieval returned the right content; the model didn't use it.

Signs:
- The retrieval span shows the expected source in top-K.
- The LLM-call span's prompt includes the source.
- The model's response doesn't reflect the source's content.

Investigation: read the assembled prompt; the source is in there but the model focused elsewhere. Check prompt instructions about using context; check for lost-in-the-middle.

Remediation: prompt engineering; context-fit selection per [retrieval-engineering.md](../rag-engineering/retrieval-engineering.md) section 6.4.

### 5.3 The "silent provider drift"

Behavior changed; investigation shows the model alias resolved to a new version.

Signs:
- `ai.llm.model_version` differs from what's pinned.
- `ai.llm.model_alias_resolved` is True.
- Many interactions show the same shift.

Investigation: model registry; check pinning.

Remediation: re-pin per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).

### 5.4 The "recent change correlation"

Quality / cost / latency shift correlates with a recent deploy.

Signs:
- Shift in trace patterns started at deploy time.
- Pre-deploy traces look different from post-deploy.

Investigation: check the deploy diff; identify the change that correlates.

Remediation: rollback the change; investigate the regression; ship the fix.

### 5.5 The "cumulative slowness"

End-to-end latency exceeds SLO; no single stage exceeds its budget.

Signs:
- Per-stage latency is each within their target.
- But the sum (or parallel-max) exceeds total budget.

Investigation: budget recalibration; the targets may not sum to the total budget correctly.

Remediation: tighten per-stage budgets; or expand total budget.

### 5.6 The "transient vs systematic"

Is this a one-off or a pattern?

Signs:
- Aggregate dashboard shows normal trends; one trace is anomalous → transient.
- Dashboard shows the shift; many traces show the pattern → systematic.

Investigation: sample multiple recent traces; compare.

Remediation: transient may need only a regression case; systematic needs a fix.

---

## 6. When re-running is appropriate

Re-running has a place; it's just not the first move.

### 6.1 Re-running for hypothesis testing

After reading the trace:

- The team has a hypothesis ("the prompt change caused this").
- Re-run with the suspected change reverted; compare results.

The replay capability per [retrieval-observability.md](../rag-engineering/retrieval-observability.md) section 8 supports this.

### 6.2 Re-running to confirm reproducibility

For intermittent issues:

- Run the same input multiple times.
- If the issue reproduces consistently: systematic.
- If it reproduces occasionally: stochastic (model nondeterminism).

The reproducibility informs the diagnosis depth.

### 6.3 Re-running for fix validation

After fix deployed:

- Re-run the originally-failing input.
- Verify the fix worked.

This validates the fix without waiting for the issue to recur naturally.

### 6.4 When re-running is wrong

- **Re-running before reading the trace.** The trace has the evidence; re-running discards it.
- **Re-running for stochastic issues without statistical analysis.** A single re-run isn't conclusive.
- **Re-running to "see what happens" without a hypothesis.** Wastes time and may not produce useful information.

The discipline: trace first; hypothesis; then re-run with purpose.

---

## 7. The diagnosis-to-fix loop

The handoff from diagnosis to remediation.

### 7.1 The diagnosis output

A complete diagnosis produces:

- Root cause identified.
- Affected scope (one interaction, one feature, many features).
- Severity assessment.
- Remediation candidate.
- Regression case description.

The diagnosis is documented (in the incident ticket or runbook log).

### 7.2 The fix path

Per the root cause:

- Prompt regression → roll back the prompt; investigate the change.
- Model regression → re-pin the model version.
- Corpus issue → trigger re-ingestion; fix the converter / chunker.
- Code bug → fix; ship with eval validation.
- Configuration error → fix the config; redeploy.

Each fix path has its own discipline (per the relevant document).

### 7.3 The regression case

Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md):

- The diagnosed bug becomes a regression case.
- The case prevents recurrence.

The fix isn't complete until the regression case is added.

### 7.4 The post-incident review

For Sev-1 / Sev-2:

- Post-mortem per the broader SRE practice.
- Observability gap analysis: was the trace adequate? Did the workflow help?
- Updates: add missing instrumentation; update the workflow; add to the failure-mode catalog.

The post-mortem improves future investigations.

---

## 8. Integration with the failure-mode catalog

The team accumulates investigation knowledge.

### 8.1 The catalog

Per [rag-failure-modes-and-debugging.md](../rag-engineering/rag-failure-modes-and-debugging.md) section 9.4:

- Each documented failure mode has its workflow.
- New patterns are added.
- The catalog grows.

The catalog is the team's accumulated investigation knowledge; new on-call engineers consult before guessing.

### 8.2 The pattern → workflow map

For each pattern in section 5:

- Documented workflow.
- Common cause.
- Remediation pattern.

The workflow is part of the on-call runbook.

### 8.3 The new-pattern protocol

When investigation surfaces a pattern that's not in the catalog:

- Document the pattern.
- Document the workflow that diagnosed it.
- Add to the catalog.

The catalog grows from operational experience.

### 8.4 The catalog review

Quarterly:

- Review recent investigations.
- Identify patterns that should be in the catalog.
- Update workflows that have evolved.

The review keeps the catalog current.

### 8.5 The cross-reference

The catalog is cross-referenced with:

- [rag-failure-modes-and-debugging.md](../rag-engineering/rag-failure-modes-and-debugging.md) for RAG-specific patterns.
- [agent-engineering-playbook.md](../agent-engineering/agent-engineering-playbook.md) for agent-specific patterns.
- This document for the broader trace-reading discipline.

The cross-references let the team navigate from "what happened" to the relevant detail.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The diagnostic toolkit

Meridian's debugging stack:

- **Datadog** for cross-system trace correlation (AI alongside the database, web tier).
- **LangSmith** for AI-specific debugging (prompt-as-debuggable; eval cross-references).
- **`meridian-replay`** CLI for hypothesis testing.

The on-call uses whichever destination fits the investigation's needs.

### 9.2 The 2026-Q2 cardiology investigation revisited

Per [chunking-engineering.md](../rag-engineering/chunking-engineering.md) section 9.5 and [retrieval-observability.md](../rag-engineering/retrieval-observability.md) section 9.4:

Full debugging walkthrough:

**Intake.**
- Alert: cardiology recall dropped from 89% to 78%.
- Time window: yesterday's afternoon to today's morning.
- Recurrence: aggregate signal; not a single trace.

**Pattern recognition.** Aggregate-not-single pattern; systematic issue.

**Trace sampling.** Pulled 10 failed-case traces from the past 6 hours.

**Top-down read across the samples.**
- All terminated normally (no budget breach, no error).
- Retrieval span: returned chunks didn't include the expected cardiology guideline chunks.

**Hypothesis.** Retrieval-stage issue per [rag-failure-modes-and-debugging.md](../rag-engineering/rag-failure-modes-and-debugging.md) section 3.2.

**Deep-dive into retrieval span.**
- Per-retriever sub-spans: cardiology chunks were retrieved at rank 25-35.
- Total candidates: 50. Top-10 passed to rerank.
- Cardiology chunks fell outside top-10.

**Hypothesis refined.** Cardiology chunks have lower retrieval scores than expected. Why?

**Inspect chunks themselves.** Pulled the affected docs from the corpus.
- Documents had been updated yesterday (AHA cardiology refresh).
- Chunks were shorter than usual (incomplete content).

**Hypothesis refined.** Chunking issue. Why are chunks short?

**Format conversion check.** Compared old vs new converter output.
- New AHA HTML included SVG figures.
- Converter's text extraction was failing on SVG-heavy sections.

**Root cause identified.** Converter doesn't handle SVG; chunks from affected sections are truncated.

**Fix.** Converter updated.

**Validation.** Used `meridian-replay` to re-run the affected interactions with the updated converter and re-ingested docs; verified retrieval restored.

**Regression case.** Added: cardiology question that specifically tests SVG-heavy guideline retrieval.

**Runbook update.** Added "check for SVG / figure handling" step in the converter diagnostic.

Total time from alert to fix-deployed: ~3 hours. Top-down trace read + selective descent identified the cause quickly.

### 9.3 The 2026-04-29 alias-drift investigation revisited

Per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) section 8.3:

**Intake.** Cost-circuit-breaker tripped on care-coordinator feature.

**Top-down read.** Per-feature cost dashboard showed care-coordinator at 100% of daily budget.

**Drill into per-prompt-version cost.** No prompt change today.

**Drill into per-model-version cost.** claude-opus-4-7 was at a different version than the pin.

**Root cause.** Model alias resolved to a new version with different pricing.

**Fix.** Re-pin to the previous specific version.

**Regression discipline.** The team's alias-banning discipline became platform-wide after this incident.

Top-down workflow caught it in ~5 minutes.

### 9.4 The pattern recognition over time

Meridian's team has built up pattern recognition:

- Quick identification of "model alias drift" (the per-model-version comparison is in the dashboard for instant inspection).
- Quick identification of "converter regression" (format-change checks early in retrieval debugging).
- Quick identification of "agent stuck in loop" (turn-count distribution in dashboard).

Each pattern was learned from an incident; codified in the runbook; faster next time.

### 9.5 The platform discipline

- Top-down workflow documented.
- Per-span debugging patterns in the runbook.
- `meridian-replay` for hypothesis testing.
- Quarterly catalog review.
- Post-incident workflow updates.

---

## 10. Anti-patterns

### 10.1 "Re-run before reading the trace"

On-call immediately reruns the question instead of reading the captured trace. Discards evidence.

**Corrective.** Trace-first workflow per section 2.

### 10.2 "Read everything"

On-call expands every span; investigation gets lost in detail.

**Corrective.** Top-down selective descent per section 3.4.

### 10.3 "Jump to assumed cause"

On-call has a hypothesis from the get-go; investigates only that; misses the real cause.

**Corrective.** Outer-to-inner read per section 3; hypothesis after observation.

### 10.4 "Single-trace conclusion"

Diagnosis based on one trace; assumes systematic; may be transient.

**Corrective.** Sample multiple traces; identify patterns per section 5.6.

### 10.5 "No regression case"

Bug fixed; no regression case; bug recurs.

**Corrective.** Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md); part of every fix.

### 10.6 "Pattern not documented"

Investigation finds a new pattern; nobody documents it; the next investigation rediscovers it.

**Corrective.** Catalog update per section 8.3.

### 10.7 "Observability gap accepted"

Investigation reveals the trace didn't have enough information; team works around it; gap persists.

**Corrective.** Post-incident observability gap analysis per section 7.4.

### 10.8 "Runbook divergence"

Multiple debugging guides exist; new on-call doesn't know which to use.

**Corrective.** Single canonical runbook structure; cross-references per section 8.5.

---

## 11. Findings (sprint-assignable)

### DBG-001 — Severity: Critical
**Finding.** No documented trace-reading workflow; investigations are ad-hoc.
**Recommendation.** Top-down workflow per section 2; documented and rehearsed.
**Owner.** ai-platform-eng team lead + sre, sprint N+1.

### DBG-002 — Severity: High
**Finding.** Investigations start with re-running; captured traces underused.
**Recommendation.** Trace-first discipline per section 2.2; enforce in runbook.
**Owner.** ai-platform-eng team lead, sprint N+1.

### DBG-003 — Severity: High
**Finding.** Per-span debugging patterns undocumented; team rediscovers each time.
**Recommendation.** Per-span patterns per section 4; documented in runbook.
**Owner.** ai-platform-eng + sre, sprint N+2.

### DBG-004 — Severity: High
**Finding.** Cross-cutting patterns not catalogued; common patterns rediscovered.
**Recommendation.** Pattern catalog per section 5; integrated with failure-mode catalog.
**Owner.** ai-platform-eng team lead, sprint N+2.

### DBG-005 — Severity: High
**Finding.** Replay capability absent; hypothesis testing requires re-running production.
**Recommendation.** Replay tool per section 6.1 / [retrieval-observability.md](../rag-engineering/retrieval-observability.md) section 8.
**Owner.** ai-platform-eng, sprint N+2.

### DBG-006 — Severity: High
**Finding.** Post-incident observability gap analysis not done; gaps persist.
**Recommendation.** Add to post-mortem template per section 7.4.
**Owner.** ai-platform-eng + sre, sprint N+2.

### DBG-007 — Severity: High
**Finding.** Time-to-diagnosis target unmet; investigations take days for Sev-2.
**Recommendation.** Workflow improvement per section 2.4; rehearse.
**Owner.** ai-platform-eng team lead, sprint N+3.

### DBG-008 — Severity: High
**Finding.** Single-trace conclusion bias; transient issues escalated to systematic.
**Recommendation.** Multi-sample analysis per section 5.6.
**Owner.** ai-platform-eng team lead, sprint N+3.

### DBG-009 — Severity: Medium
**Finding.** Catalog review not scheduled; patterns stale.
**Recommendation.** Quarterly catalog review per section 8.4.
**Owner.** ai-platform-eng team lead, sprint N+3.

### DBG-010 — Severity: Medium
**Finding.** Cross-vendor trace navigation inefficient; investigations bounce between UIs.
**Recommendation.** Per-investigation-type UI selection per section 3.6.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### DBG-011 — Severity: Medium
**Finding.** Re-running discipline absent; team re-runs whenever uncertain.
**Recommendation.** Re-running with hypothesis per section 6; trace-first first.
**Owner.** ai-platform-eng team lead, sprint N+3.

### DBG-012 — Severity: Medium
**Finding.** New patterns not promoted to the catalog; recurring rediscovery.
**Recommendation.** New-pattern protocol per section 8.3.
**Owner.** ai-platform-eng + sre, sprint N+4.

### DBG-013 — Severity: Medium
**Finding.** Diagnosis-to-fix handoff is informal; cases close without regression cases.
**Recommendation.** Per section 7; structured handoff with required regression case.
**Owner.** ai-platform-eng + sre, sprint N+4.

### DBG-014 — Severity: Medium
**Finding.** Per-feature debugging patterns conflated with cross-cutting; runbooks unclear.
**Recommendation.** Distinction per section 4 vs 5.
**Owner.** ai-platform-eng team lead, sprint N+4.

### DBG-015 — Severity: Medium
**Finding.** Visualization choice not deliberate; investigations use sub-optimal UI.
**Recommendation.** Documented per section 3.6.
**Owner.** ai-platform-eng team lead, sprint N+4.

### DBG-016 — Severity: Low
**Finding.** Investigation latency not measured; team has no SLO.
**Recommendation.** Track per section 2.4; report monthly.
**Owner.** ai-platform-eng team lead, sprint N+5.

### DBG-017 — Severity: Low
**Finding.** On-call runbook documentation thin; new on-call rotates without preparation.
**Recommendation.** Rehearsal cadence per [alerting-and-paging-design.md](./alerting-and-paging-design.md) section 6.4.
**Owner.** ai-platform-eng + sre, sprint N+5.

### DBG-018 — Severity: Low
**Finding.** Trace-reading skills not formally trained; new engineers learn by trial.
**Recommendation.** Onboarding training; structured rehearsal of common patterns.
**Owner.** ai-platform-eng team lead, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team without structured debugging:

- [ ] **Sprint 0 — workflow design.** Top-down trace-reading workflow per section 2.
- [ ] **Sprint 1 — runbook draft.** First runbooks for common patterns.
- [ ] **Sprint 1 — trace-first discipline.** Communicated; enforced in on-call review.
- [ ] **Sprint 2 — per-span debugging patterns.** Documented per section 4.
- [ ] **Sprint 2 — cross-cutting patterns.** Documented per section 5.
- [ ] **Sprint 3 — replay capability.** Hypothesis testing per section 6.
- [ ] **Sprint 3 — post-incident observability analysis.** Added to post-mortem template.
- [ ] **Sprint 4 — catalog integration.** Cross-referenced with failure-mode catalog per section 8.
- [ ] **Sprint 4 — rehearsal cadence.** Quarterly runbook walkthroughs.
- [ ] **Sprint 5 — investigation latency tracking.** SLO and reporting.
- [ ] **Ongoing — discipline.** Trace-first; new-pattern catalog updates; regression case discipline.

A team that completes this sequence turns AI debugging from "rerun and hope" into "read the trace and diagnose." Investigation latency drops; recurrence rates drop; the team's accumulated knowledge grows.

---

## 13. References

- This repo: [observability-and-telemetry/trace-and-span-design.md](./trace-and-span-design.md) — trace structure that this discipline reads.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](./llm-call-instrumentation.md) — LLM-call span attributes.
- This repo: [observability-and-telemetry/agent-step-instrumentation.md](./agent-step-instrumentation.md) — agent-turn span attributes.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](./retrieval-instrumentation.md) — retrieval span attributes.
- This repo: [observability-and-telemetry/quality-drift-detection.md](./quality-drift-detection.md) — drift signals.
- This repo: [observability-and-telemetry/cost-dashboards.md](./cost-dashboards.md) — cost dashboard for investigations.
- This repo: [observability-and-telemetry/alerting-and-paging-design.md](./alerting-and-paging-design.md) — alerts that trigger investigations.
- This repo: [observability-and-telemetry/vendor-tool-integration.md](./vendor-tool-integration.md) — UI choice across vendors.
- This repo: [rag-engineering/rag-failure-modes-and-debugging.md](../rag-engineering/rag-failure-modes-and-debugging.md) — RAG-specific patterns.
- This repo: [rag-engineering/retrieval-observability.md](../rag-engineering/retrieval-observability.md) — replay capability.
- This repo: [agent-engineering/agent-engineering-playbook.md](../agent-engineering/agent-engineering-playbook.md) — agent-specific debugging.
- This repo: [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — regression case workflow.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — architecture context.
