# Eval of RAG

> **Audience.** Engineers building or refactoring the eval suite for a RAG system. Tech leads who have been asked "why did the model say that?" and want a structured answer beyond "we re-ran and got something different." **Scope.** The *engineering* practice of RAG-specific eval — separating retrieval failures from generation failures, scoring citation accuracy and faithfulness, RAG-specific case design. Pair with [eval-engineering-playbook.md](./eval-engineering-playbook.md) (the broader eval practice). Not the architectural RAG-pattern decision (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `reference-patterns/rag-architecture-decision-guide.md`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Generic answer-quality eval treats the system as a black box: "the answer is right or wrong." For RAG systems, this misses the dominant diagnostic question: *was the answer wrong because retrieval returned the wrong content, or because the model misused the right content?* Without separation, every quality regression looks the same; the team cannot tell what to fix.

The RAG-eval discipline introduces stage-specific scoring: retrieval recall (was the right source returned?), retrieval precision (were the returned sources relevant?), citation accuracy (does the cited source support the claim?), faithfulness (is the answer grounded in the retrieved content?), empty-retrieval handling (did the system handle no-results gracefully?). Each is independently measurable; together they decompose RAG quality into actionable components.

The RAG decision guide's `ARCH-RAG-013` finding is the canonical instance — without stage separation, RAG diagnosis is guesswork. This document closes that finding from the engineering side.

This document is opinionated about three things:

1. **Retrieval and generation are scored separately.** A failing case decomposes into retrieval-stage failure and/or generation-stage failure. The two have different fixes (retriever tuning vs prompt tuning).
2. **Citation accuracy is its own metric.** Whether the model's claim is supported by the cited source is independently measurable. Hallucinated citations are the most common RAG failure mode in regulated workloads.
3. **Empty-retrieval handling is part of the contract.** When retrieval returns nothing, the system's behavior matters. "Make something up" is wrong; "I do not know" is right.

Structure: (2) the RAG eval taxonomy; (3) golden-set design for RAG; (4) the stage-specific eval; (5) citation accuracy and faithfulness; (6) empty-retrieval cases; (7) the LLM-as-judge for RAG; (8) integration with the broader eval practice; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The RAG eval taxonomy

RAG quality decomposes into several dimensions, each independently scoreable.

### 2.1 Retrieval-stage dimensions

- **Recall@K.** Was the gold-standard source in the top-K retrieved chunks? (Where K is the configured top-K.)
- **Recall by source.** When multiple gold sources are expected, what fraction were retrieved?
- **Precision@K.** Of the top-K retrieved, what fraction were actually relevant?
- **MRR (Mean Reciprocal Rank).** Where in the ranking did the gold source appear?
- **Empty-retrieval rate.** What fraction of cases returned zero results?

### 2.2 Generation-stage dimensions

- **Answer correctness.** Was the final answer correct (given the retrieved context)?
- **Citation accuracy.** Did cited claims actually appear in the cited source?
- **Faithfulness.** Was the answer grounded in the retrieved content (no unsupported claims)?
- **Format adherence.** Did the answer follow the expected format?
- **Refusal correctness.** When the model refused, was refusal appropriate?

### 2.3 End-to-end dimensions

- **Overall case pass.** Did the case pass on all dimensions?
- **User-facing quality.** How would a user rate this answer?

### 2.4 Why decomposition matters

When a case fails on overall quality:

| Retrieval recall | Citation accuracy / answer correctness | Diagnosis |
|---|---|---|
| Pass | Pass | Case passed |
| Pass | Fail | Generation problem: the right context was available; the model failed to use it correctly |
| Fail | (Pass or Fail) | Retrieval problem: the right context was not available; generation could not succeed |

The decomposition tells the team what to fix. Without it, every failure is "improve quality" — a directionless instruction.

---

## 3. Golden-set design for RAG

The RAG eval case requires additional fields beyond a generic eval case.

### 3.1 The RAG case structure

```yaml
id: RAG-GOLD-0014
question: "What is the post-discharge follow-up protocol for a CHF patient on the new pathway?"

# Expected sources — the chunks/docs that should be retrieved
expected_sources:
  - chunk_id: "clinical-guideline:aha-hf-2024:section-3.2:chunk-0042"
    document_id: "clinical-guideline:aha-hf-2024:section-3.2"
    is_required: true
    description: "AHA 2024 HF discharge bundle protocol"
  - chunk_id: "tenant-protocol:mercy-cleveland:hf-22:chunk-0007"
    document_id: "tenant-protocol:mercy-cleveland:hf-22"
    is_required: true
    description: "Mercy Cleveland HF-22 protocol"

# Expected answer — semantic reference; not exact match
expected_answer: |
  Post-discharge follow-up for a CHF patient on the new pathway:
  - 7-day nursing check-in
  - 14-day clinician visit
  Citations: AHA 2024 HF Discharge Bundle section 3.2; Mercy Cleveland Protocol HF-22.

# Required claims — claims the answer must include with citation
required_claims:
  - claim: "7-day nursing check-in is required"
    must_cite: "clinical-guideline:aha-hf-2024:section-3.2"
  - claim: "14-day clinician visit is required"
    must_cite: "clinical-guideline:aha-hf-2024:section-3.2"

# Per-stage scoring rubric
scoring:
  retrieval:
    recall_at_K: 10
    minimum_required_recall: 1.0  # all required expected_sources must be retrieved
  generation:
    answer_correctness: must_include_all_required_claims
    citation_accuracy: each_required_claim_must_be_correctly_cited
    faithfulness: must_not_make_claims_outside_retrieved_content

# Metadata
case_class: "clinical-protocol"
tenant_context:
  tenant_id: "mercy-cleveland"
  caller_role: "rn"
```

### 3.2 The required vs optional sources

`is_required: true` means the source must be retrieved for the case to pass on retrieval-recall.

`is_required: false` (or omitted) means the source is acceptable if retrieved but not strictly required (multiple equivalent sources exist; any one is sufficient).

The distinction supports cases where multiple valid sources can answer the question.

### 3.3 The required-claims field

The required-claims field is what supports citation accuracy and faithfulness scoring:

- For each required claim, the answer must include it.
- For each required claim with `must_cite`, the citation must point to the named source.

This is more rigorous than a generic "is the answer right." The model must say the right things AND cite the right sources.

### 3.4 The tenant-context field

For multi-tenant systems, the case's tenant context determines the retrieval scope. The retrieval is performed in the tenant's context; tenant-specific results (e.g., the Mercy Cleveland protocol) are expected.

Cases that don't specify tenant context test general / global content.

### 3.5 The case-class taxonomy

Per [eval-engineering-playbook.md](./eval-engineering-playbook.md), cases are tagged with classes. For RAG, useful classes:

- By question type: lookup, definition, multi-hop, comparison.
- By corpus: clinical-guidelines, tenant-protocols, drug-interactions.
- By complexity: simple-retrieval (one chunk), multi-source (two-plus chunks), reasoning-required.
- By tenant scope: global-only, tenant-only, mixed.

Subset analysis surfaces where regressions are occurring.

---

## 4. The stage-specific eval

The eval runner produces per-stage scores.

### 4.1 The eval flow

```
For each case:
  1. Execute retrieval with the case's tenant context and the case's question.
  2. Score retrieval:
     - Did the top-K include all required expected_sources? (recall_at_K)
     - What fraction of returned chunks are in expected_sources? (precision_at_K)
     - Where in the ranking did each expected_source appear? (MRR)
  3. Pass the retrieval results to the generation step.
  4. Score generation:
     - Does the answer include each required_claim? (answer_correctness)
     - Are the citations on required_claims correct? (citation_accuracy)
     - Are all claims in the answer grounded in the retrieved chunks? (faithfulness)
  5. Compute overall pass: all dimensions must pass.
```

### 4.2 The per-stage report

For each case, the report shows:

```yaml
case_id: RAG-GOLD-0014
overall_pass: false
retrieval:
  pass: true
  recall_at_10: 1.0  # both required sources retrieved
  precision_at_10: 0.2  # 2 of 10 returned were expected; rest were related but not required
  expected_sources_ranks: [3, 5]  # required sources appeared at ranks 3 and 5
generation:
  pass: false
  answer_correctness: false  # missed the 14-day clinician visit claim
  citation_accuracy: true  # citations on what was claimed were correct
  faithfulness: true  # no unsupported claims
  diagnosis: "Answer included 7-day check-in claim correctly but omitted the 14-day clinician visit claim."
```

The diagnosis points to the specific failure: generation missed a required claim that was retrievable in the context. The fix is in the generation prompt (instruct the model to enumerate all relevant follow-up timepoints) or in the retrieval (the chunk containing the 14-day claim was retrieved at rank 5, near the end of the top-K — could be re-ranked higher).

### 4.3 The aggregate scoring

Across the eval suite:

| Metric | Value |
|---|---|
| Overall pass rate | 87% |
| Retrieval recall@10 pass rate | 95% |
| Retrieval precision@10 mean | 0.32 |
| Generation answer-correctness (given correct retrieval) | 92% |
| Citation accuracy | 98% |
| Faithfulness | 96% |
| Empty-retrieval rate | 1.8% |

The decomposition makes the priorities visible: retrieval is mostly fine (95% recall); generation has the larger gap (92% on cases with correct retrieval); citation accuracy is high; faithfulness is high.

### 4.4 The drill-down per case class

Aggregate by class:

| Case class | Overall | Retrieval recall | Generation correctness |
|---|---|---|---|
| Simple-retrieval (clinical) | 95% | 99% | 96% |
| Multi-source (clinical) | 78% | 92% | 85% |
| Multi-hop (clinical) | 65% | 80% | 81% |
| Drug-interaction | 91% | 96% | 95% |

The multi-hop class shows the lowest recall AND the lowest generation correctness — the team's priority is improving multi-hop handling (likely via agentic RAG or query rewriting per the architecture sibling's `reference-patterns/rag-architecture-decision-guide.md`).

---

## 5. Citation accuracy and faithfulness

These two dimensions are RAG-specific and underused.

### 5.1 Citation accuracy

For each required claim, check:
- Is the claim cited?
- Does the cited source actually contain content supporting the claim?

The judge LLM is given: the claim, the cited source content. Asked: does the source support the claim?

Citation accuracy is the foundation for the user-facing citation feature. If the citation accuracy is below threshold, the citations cannot be trusted; the user-facing citation links become hazards rather than affordances.

### 5.2 Faithfulness

For each claim in the answer (cited or not), check:
- Is the claim supported by content in the retrieved chunks?

The judge LLM is given: the claim, the full retrieved-chunk-set. Asked: is the claim grounded in any of the chunks?

Faithfulness catches hallucinated claims — content the model produced that has no grounding in retrieval. For regulated workloads, hallucinated claims are unacceptable.

### 5.3 The judge prompt for citation accuracy

```
You are evaluating whether a clinical claim is supported by a cited source.

CLAIM:
{claim}

CITED SOURCE CONTENT:
{source_content}

QUESTION:
Does the cited source content support the claim?

Return JSON:
{
  "supported": true | false,
  "justification": "..."
}
```

The judge runs per-claim. Aggregated, the citation-accuracy metric is the fraction of cited claims that the judge ratifies as supported.

### 5.4 Calibration

Citation-accuracy and faithfulness judges are themselves eval-validated:
- Sample 30-50 judge decisions; human (clinical reviewer) checks each.
- Inter-rater agreement between judge and human computed.
- Judge prompt revised if agreement is below 85%.

Recalibration quarterly per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 8.

### 5.5 The hallucinated-citation pattern

The most common citation failure: the model cites a source that does not actually contain the claim. The model "knows" the claim from training data and cites the retrieved source as confirmation. The retrieved source is something else; the citation is hallucinated.

Faithfulness scoring catches this. The hallucinated citation appears as a citation accuracy failure AND a faithfulness pass (the claim is not in the retrieved content, so it cannot be supported there).

---

## 6. Empty-retrieval cases

When retrieval returns zero relevant results, the system's behavior is important.

### 6.1 The expected behavior

The model should:
- Not fabricate an answer.
- Acknowledge that retrieval did not find relevant content.
- For regulated workloads: refuse or escalate, with clarity about the limitation.

For Meridian: the clinical-knowledge worker explicitly refuses to answer clinical questions when retrieval returns nothing relevant; the response is "I don't have information on that in the available clinical content; please consult an authoritative source."

### 6.2 The eval case

Empty-retrieval cases in the eval suite:

```yaml
id: RAG-EMPTY-001
question: "What is the dosing for [intentionally unknown drug name]?"
expected_behavior: refuse_with_unknown_drug_message
expected_sources: []  # empty by design

scoring:
  must_refuse: true
  must_not_fabricate: true
  must_mention_limitation: true
```

The case tests that the system handles the empty case gracefully.

### 6.3 The dangerous case

The dangerous case: the model produces an answer despite no relevant retrieval. Either the model is using training knowledge (acceptable if disclosed, dangerous if undisclosed) or the model is hallucinating (always wrong).

The eval suite has empty-retrieval cases that test this. The judge prompt looks for: did the model refuse appropriately, OR did the model produce an answer with explicit "I don't have this in the retrieved content" disclosure.

### 6.4 The aggregate empty-retrieval handling

The system's empty-retrieval-handling pass rate is itself a metric. Below threshold, the team knows the system is producing answers in cases it should not.

---

## 7. The LLM-as-judge for RAG

The judge architecture per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 4 applies, with RAG-specific extensions.

### 7.1 The judge's task

For each case, the judge receives:
- The question.
- The retrieved chunks (for faithfulness and per-chunk scoring).
- The expected sources (for retrieval scoring).
- The required claims (for citation accuracy and answer correctness).
- The produced answer.

The judge returns:
- Per-dimension pass/fail.
- Justification per dimension.
- Overall pass/fail.

### 7.2 The composite judge

For comprehensive scoring, the judge may run multiple sub-judges:

- Citation accuracy judge (per-claim).
- Faithfulness judge (per-claim).
- Answer correctness judge (overall).
- Refusal correctness judge (when applicable).

Each sub-judge is calibrated independently. Composition is the engineering work of running each and aggregating.

### 7.3 The judge model selection

The judge should be more capable than the production system:
- Production system on Sonnet → judge on Opus.
- Production system on Opus → judge on Opus (different prompt, larger context budget) or a specialized eval-tuned model.

The judge's calibration validates that the judge agrees with human review on a sample.

### 7.4 The judge's cost

RAG eval is expensive at the per-case level (multiple LLM calls per case). For a 200-case suite with full judge coverage:
- ~200 retrieval scoring calls + ~200 answer correctness calls + ~600 citation/faithfulness calls (per-claim) = ~1000 judge calls per suite run.
- At judge-tier pricing: ~$5-15 per suite run.

The cost is justified by the eval discipline's value, but it caps full-suite frequency. Run the full suite nightly + on release candidates; use stratified subset for per-PR checks.

---

## 8. Integration with the broader eval practice

RAG eval is a specialization of the broader eval practice. The integration:

### 8.1 The eval framework

RAG eval cases live in the same eval suite as non-RAG cases. The eval runner detects RAG-specific fields (expected_sources, required_claims) and applies RAG-specific scoring.

The pass/fail and aggregate metrics are reported in the same dashboards as general eval results.

### 8.2 The CI integration

Per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 5, eval gates CI. RAG-specific thresholds:

- Per-PR: stratified subset includes RAG cases proportional to their share of the workload.
- Nightly: full RAG suite.
- Release candidate: full RAG suite + fresh corpus retrieval-recall check (sample of production queries).

### 8.3 The regression discipline

When a production RAG bug is fixed, the case enters the regression suite per section 6 of the playbook. RAG cases in the regression suite:

- The exact question that failed.
- The retrieval results captured at the time of the failure.
- The wrong answer (for diagnostic context).
- The corrected expected answer / sources / claims.

The regression suite documents the specific failure modes the team has observed and prevents recurrence.

### 8.4 The online judge for RAG

Per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 7.3, online judge samples production traffic. For RAG, the online judge runs:

- Retrieval recall on a sample (where expected sources are known).
- Citation accuracy on every sampled response.
- Faithfulness on every sampled response.

Online judge results feed the production quality SLI.

### 8.5 The lineage integration

Citation accuracy depends on the lineage from answer → cited source → source content (per [lineage-and-provenance.md](../../ai-architecture-reference-architecture/data-architecture-for-ai/lineage-and-provenance.md)). The eval runner consumes the lineage; without it, citation validation requires reconstruction.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The eval suites

The Care Coordinator's RAG-relevant suites:

| Suite | Cases | Cadence | RAG-specific |
|---|---|---|---|
| Clinical golden set | 200 | Quarterly + incident-driven | Yes — uses RAG case structure |
| Drug-interaction subset | 60 | Quarterly + on FDA changes | Yes — drug-interaction graph augmented |
| Conversational subset | 50 multi-turn | Quarterly | Yes — RAG context per turn |
| Refusal / escalation subset | 30 | Quarterly | Includes empty-retrieval cases |
| Regression | growing (~90) | Per-bug-fix | Yes — RAG cases inherit RAG structure |

Each suite has the RAG-specific fields (expected_sources, required_claims) for cases that use retrieval.

### 9.2 The eval runner

`meridian.eval.rag_runner` is the RAG-specific eval runner. It:

1. Loads cases from the suite YAML files.
2. For each case: executes retrieval (through the production retrieval client, with eval-tenant context); captures returned chunks.
3. Scores retrieval (recall, precision, MRR per case).
4. Generates the answer (through the production LLM client, with eval-prompt-version pinned).
5. Scores generation via the judge (answer correctness, citation accuracy, faithfulness).
6. Aggregates and reports.

The runner produces a detailed per-case report and an aggregate pass-rate.

### 9.3 The pass-rate dashboard

| Suite | Overall pass rate (current) | Retrieval recall | Generation correctness | Citation accuracy | Faithfulness |
|---|---|---|---|---|---|
| Clinical golden set | 95.2% | 96% | 97% (given correct retrieval) | 99% | 98% |
| Drug-interaction | 98.1% | 99% | 99% | 99.5% | 99% |
| Conversational | 91.5% | 86% (turn 2+) | 95% | 98% | 97% |
| Refusal / escalation | 96.7% | n/a | n/a | n/a | n/a |

The decomposition shows: clinical is well-balanced; drug-interaction is consistently high; conversational has retrieval recall as the gap (the query rewriter helps but not fully); refusal is mostly working.

### 9.4 A diagnostic walkthrough

In 2026-Q1, the clinical golden set's pass rate dropped 4 points (95% → 91%). The diagnostic:

1. Per-case-class breakdown showed the regression was concentrated in the cardiology subset (down 8 points).
2. Per-stage breakdown showed retrieval recall held (96% → 95%, within noise) but generation correctness dropped (96% → 88%).
3. Sample of failing cases showed the model was correctly retrieving cardiology guidelines but ignoring a key paragraph about contraindications.
4. Inspection of the production trace showed the cardiology guidelines were being retrieved at rank 6-9 of top-10 (consistent with the ARCH-RAG-013 reranker prompt issue described in [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) section 9.4).
5. Fix: reranker prompt rolled back. Eval re-run showed pass rate restored.

The decomposition turned what could have been "improve clinical quality" into the specific finding: generation issue, due to retrieval ranking, due to reranker prompt change.

### 9.5 The regression case

After the 2026-Q1 fix, a regression case was added:

```yaml
id: RAG-REG-0087
question: "What are the contraindications for ACE inhibitors in heart-failure patients?"
expected_sources:
  - chunk_id: "clinical-guideline:aha-hf-2024:section-5.4:chunk-0021"
    is_required: true
required_claims:
  - claim: "ACE inhibitors are contraindicated in patients with renal artery stenosis"
    must_cite: "clinical-guideline:aha-hf-2024:section-5.4"
scoring:
  retrieval:
    expected_sources_must_appear_in_top: 5  # tighter than default top-10
notes: |
  Original failure (2026-Q1): the reranker prompt change pushed this guideline
  to rank 8; generation missed the contraindication. Eval gate now requires
  the contraindication source at rank 5 or higher.
```

The case prevents the specific failure from recurring; the tighter rank-5 requirement is the engineering response to the observed weakness.

### 9.6 The online judge

10% of production Care Coordinator interactions are sampled for online judge runs. The online judge:

- Verifies citation accuracy on every sampled response (where citations were produced).
- Verifies faithfulness on every sampled response.
- The judge-pass-rate is the production quality SLI.

When the SLI drops > 8 points in a 4-hour window, the Tier 1 alert fires (per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md)).

### 9.7 The platform discipline

- Every RAG case in the golden set has the full structure (expected_sources, required_claims).
- The eval runner separates retrieval and generation scoring.
- The aggregate dashboard shows per-stage metrics; investigators can drill down.
- The regression suite documents every observed failure mode.
- Online judge runs continuously sample production for citation accuracy and faithfulness.
- Quarterly recalibration of the judge prompts against clinician review.

---

## 10. Anti-patterns

### 10.1 "Single overall-pass metric"

The eval reports overall case pass/fail. When pass rate drops, the team cannot tell whether retrieval or generation is the issue.

**Corrective.** Per-stage scoring per section 4. Decomposition is the diagnostic power.

### 10.2 "Citation accuracy not measured"

The team checks that the answer is right but does not verify that citations point to content actually supporting the claim. Hallucinated citations ship.

**Corrective.** Citation accuracy as a dimension per section 5; required-claims with must_cite in case structure.

### 10.3 "Faithfulness not measured"

The team measures answer correctness against the expected answer but not whether the answer is grounded in retrieval. The model can produce a correct answer that is not actually supported by the retrieved content (a coincidence) — and a future change to retrieval will break it.

**Corrective.** Faithfulness as a dimension; per-claim grounding check.

### 10.4 "Empty-retrieval cases absent"

The eval suite has no cases where retrieval should return zero results. The system's empty-retrieval behavior is unverified; the model may be fabricating answers in empty-retrieval cases without detection.

**Corrective.** Empty-retrieval cases per section 6 — at least 5-10% of the suite for regulated workloads.

### 10.5 "Expected sources not in case structure"

The case has expected_answer but not expected_sources. Retrieval recall cannot be measured; the per-stage decomposition is impossible.

**Corrective.** Expected sources per case per section 3.

### 10.6 "Judge is the same model as production"

The judge is on Sonnet; production is on Sonnet. The judge agrees with its own (production) mistakes. Quality issues are missed.

**Corrective.** Judge on a more capable tier per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 4.2.

### 10.7 "No regression cases from production"

Production RAG bugs are fixed but no regression case is added. The bug recurs (sometimes years later) when an unrelated change triggers the same condition.

**Corrective.** Bug-fix → regression case process per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 6.

### 10.8 "Online judge not running"

The team has offline eval; production quality is unmeasured continuously. Quality regressions in production are detected by user complaints.

**Corrective.** Online judge per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 7.3.

---

## 11. Findings (sprint-assignable)

### EVAL-RAG-001 — Severity: Critical
**Finding.** RAG eval produces single overall pass/fail; retrieval failures and generation failures cannot be distinguished.
**Recommendation.** Per-stage scoring per section 4; expected_sources in case structure.
**Owner.** ai-platform-eng, sprint N+1.

### EVAL-RAG-002 — Severity: Critical
**Finding.** Citation accuracy is not measured; hallucinated citations ship to users.
**Recommendation.** Required-claims with must_cite in case structure; judge for citation accuracy per section 5.1.
**Owner.** ai-platform-eng, sprint N+1.

### EVAL-RAG-003 — Severity: Critical
**Finding.** Faithfulness is not measured; the model can produce ungrounded claims without detection.
**Recommendation.** Faithfulness judge per section 5.2; per-claim grounding check.
**Owner.** ai-platform-eng, sprint N+2.

### EVAL-RAG-004 — Severity: High
**Finding.** Empty-retrieval cases are absent from the suite; system behavior on empty results is unverified.
**Recommendation.** Add empty-retrieval cases per section 6; cover dangerous-fabrication scenarios.
**Owner.** ai-platform-eng + clinical-domain-experts, sprint N+2.

### EVAL-RAG-005 — Severity: High
**Finding.** Expected sources are not in case structure; retrieval recall cannot be measured.
**Recommendation.** Add expected_sources to every RAG case per section 3.
**Owner.** ai-platform-eng, sprint N+2.

### EVAL-RAG-006 — Severity: High
**Finding.** Judge model is the same as production model; judge agrees with production mistakes.
**Recommendation.** Judge on more capable tier per section 7.3.
**Owner.** ai-platform-eng, sprint N+2.

### EVAL-RAG-007 — Severity: High
**Finding.** RAG-specific dimensions are not surfaced in dashboards; aggregate quality is reported but per-dimension breakdown is missing.
**Recommendation.** Per-dimension dashboards per section 4.4.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### EVAL-RAG-008 — Severity: High
**Finding.** Per-case-class breakdown is missing; regressions in specific classes are not surfaced.
**Recommendation.** Case-class taxonomy per section 3.5; aggregate by class.
**Owner.** ai-platform-eng, sprint N+3.

### EVAL-RAG-009 — Severity: High
**Finding.** Online judge is not running; production RAG quality is not continuously monitored.
**Recommendation.** Sampling per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 7.3.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### EVAL-RAG-010 — Severity: Medium
**Finding.** Citation-accuracy judge is not calibrated against human review.
**Recommendation.** Calibration per section 5.4; quarterly recalibration.
**Owner.** ai-platform-eng + clinical-domain-experts, sprint N+3.

### EVAL-RAG-011 — Severity: Medium
**Finding.** Regression cases from production bugs are not added to the suite.
**Recommendation.** Bug-fix → regression-case process per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 6.
**Owner.** ai-platform-eng, sprint N+3.

### EVAL-RAG-012 — Severity: Medium
**Finding.** Multi-turn (conversational) RAG eval does not separate per-turn retrieval and generation scoring.
**Recommendation.** Per-turn scoring for multi-turn cases; capture per-turn expected_sources.
**Owner.** ai-platform-eng, sprint N+3.

### EVAL-RAG-013 — Severity: Medium
**Finding.** Lineage integration is absent; citation accuracy scoring requires reconstruction from production traces.
**Recommendation.** Consume lineage per [lineage-and-provenance.md](../../ai-architecture-reference-architecture/data-architecture-for-ai/lineage-and-provenance.md); judge consumes lineage directly.
**Owner.** ai-platform-eng, sprint N+4.

### EVAL-RAG-014 — Severity: Medium
**Finding.** Eval suite does not include adversarial / prompt-injection cases for RAG (poisoned source documents).
**Recommendation.** Adversarial subset jointly with [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture).
**Owner.** ai-platform-eng + ai-security, sprint N+4.

### EVAL-RAG-015 — Severity: Medium
**Finding.** Per-case judge cost is meaningful at suite size; suite runs are expensive.
**Recommendation.** Stratified-subset for per-PR runs; full suite nightly; cost-aware sampling for online judge.
**Owner.** ai-platform-eng + finops, sprint N+4.

### EVAL-RAG-016 — Severity: Low
**Finding.** Refusal-correctness for empty-retrieval cases is not separately scored.
**Recommendation.** Refusal-correctness as a dimension; aggregate metric.
**Owner.** ai-platform-eng, sprint N+5.

### EVAL-RAG-017 — Severity: Low
**Finding.** Multi-source cases (multiple required sources) do not have partial-credit scoring.
**Recommendation.** Partial-credit scoring for cases where some required sources retrieved; aggregate per-source-recall.
**Owner.** ai-platform-eng, sprint N+5.

### EVAL-RAG-018 — Severity: Low
**Finding.** Eval suite documentation is thin; new contributors do not understand the RAG case structure.
**Recommendation.** Documentation generated from the case schema; commit alongside the suite.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team with generic eval but not RAG-specific:

- [ ] **Sprint 0 — design.** Define the RAG case structure per section 3. Identify the scoring dimensions needed for the workload.
- [ ] **Sprint 1 — augment existing cases.** Add expected_sources and required_claims to existing RAG cases.
- [ ] **Sprint 1 — per-stage scoring.** Eval runner produces per-stage scores per section 4.
- [ ] **Sprint 2 — citation accuracy judge.** Build the judge for citation accuracy per section 5; calibrate.
- [ ] **Sprint 2 — faithfulness judge.** Build the judge for faithfulness; calibrate.
- [ ] **Sprint 3 — empty-retrieval cases.** Add cases per section 6.
- [ ] **Sprint 3 — case-class taxonomy.** Tag cases with classes; per-class dashboards.
- [ ] **Sprint 4 — online judge.** Sample production; citation accuracy and faithfulness continuously.
- [ ] **Sprint 4 — regression discipline.** Bug fixes add regression cases.
- [ ] **Sprint 5 — adversarial subset.** Coordination with security for RAG-specific adversarial coverage.
- [ ] **Ongoing — discipline.** Quarterly calibration; regression case maintenance; suite growth from production traffic.

A team that completes this sequence has the RAG-eval discipline that makes RAG quality investigations actionable. A team without per-stage scoring debugs RAG by guessing; this discipline removes the guesswork.

---

## 13. References

- This repo: [eval-engineering/eval-engineering-playbook.md](./eval-engineering-playbook.md) — the broader eval practice.
- This repo: [eval-engineering/golden-set-design.md](./) (coming) — golden-set design depth.
- This repo: [eval-engineering/llm-as-judge-patterns.md](./) (coming) — judge patterns depth.
- This repo: [eval-engineering/online-eval-and-feedback.md](./) (coming) — online judge patterns.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — the retrieval observability that informs eval design.
- Sibling repo: [ai-architecture-reference-architecture/reference-patterns/rag-architecture-decision-guide.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-patterns/rag-architecture-decision-guide.md) — ARCH-RAG-013 is the cross-link finding this closes.
- Sibling repo: [ai-architecture-reference-architecture/data-architecture-for-ai/lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md) — lineage that supports citation accuracy scoring.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the architecture context.
- Sibling repo: [ai-security-reference-architecture/llm-application-security/](https://github.com/jeremiahredden/ai-security-reference-architecture) — adversarial RAG patterns.
- RAGAS, TruLens, Phoenix — eval framework documentation that implements many of these patterns.
