# RAG Failure Modes and Debugging

> **Audience.** RAG on-call engineers and tech leads. Anyone who has been paged for "the model gave a wrong answer" and needs a structured diagnostic playbook. **Scope.** The *engineering* practice of diagnosing production RAG failures — failure-mode catalog, diagnostic workflows per mode, remediation patterns. The on-call runbook for RAG. Pair with [retrieval-observability.md](./retrieval-observability.md), [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

RAG fails in many distinguishable ways. "The model gave a wrong answer" is the symptom; the cause could be in retrieval, generation, corpus, ingestion, embedding, prompt, model, configuration. Without a structured diagnostic discipline, every investigation starts from scratch; the team takes hours to find the cause; recurrences happen because the team forgets what they learned.

The discipline this document codifies: a catalog of distinguishable failure modes, each with a documented diagnostic workflow and remediation pattern. On-call engineers walk through the diagnostic steps; the cause is identified; the remediation follows.

This document is opinionated about three things:

1. **The trace is the primary evidence.** The captured trace (per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) and [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md)) is what investigations read. Re-running is a secondary tool.
2. **Failure modes are catalogued.** Each has its own diagnostic pattern. The team accumulates the catalog over time.
3. **Every diagnosed failure becomes a regression case.** Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md), the bug-fix workflow adds a case to prevent recurrence.

Structure: (2) the failure-mode taxonomy; (3) retrieval-stage failures; (4) generation-stage failures; (5) corpus-stage failures; (6) configuration-stage failures; (7) the diagnostic workflow framework; (8) remediation patterns; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The failure-mode taxonomy

RAG failures fall into stage-specific classes. Each stage produces its own kinds of failure.

### 2.1 The stage map

| Stage | Failure class |
|---|---|
| Ingestion | Document not in corpus; document corrupted in ingestion; document stale |
| Chunking | Chunk split mid-content; chunk size wrong; structural metadata lost |
| Embedding | Embedding from wrong model version; embedding corrupted (NaN); embedding from old corpus |
| Retrieval | Top-K missing expected chunk; wrong filter applied; wrong scope |
| Reranking | Rerank dropped good chunk; rerank wrong ordering |
| Query handling | Query rewriting wrong; query rewriting hallucinated context |
| Generation | Model misused correct context; model hallucinated unsupported claim; model wrong citation |
| Configuration | Wrong model version; wrong prompt version; wrong eval suite |
| Observability | Missing instrumentation; misleading metrics |

The taxonomy guides diagnosis: the on-call asks "which stage?" before "what specifically?"

### 2.2 The user-visible symptom vs root cause

The user reports one of a few symptoms:

- Wrong answer.
- Empty / "I don't know" response.
- Off-topic answer.
- Slow response.
- Inconsistent answer (different each time).

Each symptom can be caused by failures across multiple stages. The diagnostic narrows from symptom to stage to specific cause.

### 2.3 The diagnostic shape

```
Symptom (from user report or alert)
    │
    ▼
Identify the affected interaction (interaction ID).
    │
    ▼
Pull the trace.
    │
    ▼
Read the trace stage-by-stage; identify which stage failed.
    │
    ▼
Deep-dive into the failing stage.
    │
    ▼
Identify root cause.
    │
    ▼
Apply remediation.
    │
    ▼
Add regression case.
```

The workflow is consistent across failure modes; the specifics vary per stage.

---

## 3. Retrieval-stage failures

Failures where retrieval returned the wrong content.

### 3.1 The empty-retrieval failure

**Symptom.** Retrieval returned zero results.

**Diagnostic.**
1. Trace: confirm retrieval span shows `returned_count: 0`.
2. Was the filter too restrictive? (Inspect `tenant_filter`, `content_type` filter; manually check if matching content exists.)
3. Did the query match nothing? (Try variations; check if the corpus contains relevant content.)
4. Was the embedding malformed? (Check embedding-span attributes; NaN, dimension mismatch.)
5. Was the corpus empty for this tenant?

**Common causes.**
- Filter is too restrictive (wrong tenant, wrong content-type).
- Query is too narrow (the corpus has related content but not exact).
- Corpus has no relevant content (legitimate empty; the system should refuse).
- Retrieval pipeline error (BM25 down, vector store down).

**Remediation.** Depends on cause. Add eval case for the input that produced empty. If the system should have refused, ensure the empty-retrieval handling per [eval-of-rag.md](../eval-engineering/eval-of-rag.md) section 6.

### 3.2 The missed-expected-source failure

**Symptom.** The model didn't use the expected source; investigation shows the source wasn't retrieved.

**Diagnostic.**
1. Trace: inspect `doc_ids` returned by retrieval.
2. Was the expected source in the corpus?
3. Per-retriever sub-spans: was the source in BM25 top-50? In vector top-50?
4. Was the source filtered out by the merge or rerank?
5. Was the source's chunk in a position past top-K?

**Common causes.**
- Source was in retrieval but didn't make the top-K (recall issue; consider increasing top-K or improving merge).
- Source's chunk has low embedding similarity to the query (chunking issue per [chunking-engineering.md](./chunking-engineering.md)).
- Source was filtered out (per-record scope; sensitivity scope).
- Source was rerranked down (rerank-quality issue per [reranking-engineering.md](./reranking-engineering.md)).

**Remediation.** Tune retrieval (hybrid weights, candidate counts); investigate chunking; add regression case.

### 3.3 The wrong-retrieval failure

**Symptom.** The model used the retrieved content but the content didn't actually contain the answer; the model misinterpreted.

**Diagnostic.**
1. Trace: inspect doc_ids returned; read the actual chunk content.
2. Was the retrieved content semantically similar to the query but not actually answering it?
3. Did the query have multiple valid interpretations?
4. Was there a better-matching chunk in the corpus that didn't rank as high?

**Common causes.**
- Embedding similarity captures topical similarity but not specific-match (a chunk about "drug X dosing in general" embedded similar to "drug X dosing in elderly" but doesn't answer the elderly question).
- Query ambiguity (the model picked the wrong interpretation).
- Reranker preferred the wrong chunk.

**Remediation.** Query rewriting (clarify the query); reranking (the reranker should prefer specific over general); add regression case.

### 3.4 The stale-retrieval failure

**Symptom.** Retrieval returned content that's out of date.

**Diagnostic.**
1. Trace: inspect doc_ids; check the source documents' `last_modified` timestamps.
2. Was the corpus updated since the ingest? (Check ingestion run history.)
3. Was the source updated upstream but the ingest didn't catch it? (Check connector logs.)
4. Was the content removed upstream but still in the corpus? (Check deduplication / cleanup.)

**Common causes.**
- Ingestion pipeline is behind schedule.
- Upstream change-detection missed an update.
- Source document was updated; the embedding pipeline hasn't re-embedded yet.
- Cache is serving stale content (if caching is in use).

**Remediation.** Trigger ingestion; investigate change-detection; cache invalidation; add freshness SLO monitoring.

### 3.5 The scope-violation failure (Sev-1)

**Symptom.** Retrieved content belongs to a different tenant than the requesting context.

This is a Sev-1 isolation failure per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md). The detection should have caught it before delivery to the model; if it didn't, that's a pipeline bug too.

**Diagnostic.**
1. Confirm via trace: the returned chunk's `tenant_id` doesn't match the requesting `tenant_id`.
2. Was the wrapper's mandatory filter applied?
3. Was the post-retrieval verification skipped or failed silently?

**Remediation.** Per the scope-enforcement document's incident response. Block the affected tenant pair from cross-retrieval; investigate the bypass; deploy a fix; post-mortem.

### 3.6 The retrieval-error failure

**Symptom.** Retrieval threw an error (timeout, connection failure, API error).

**Diagnostic.**
1. Trace: confirm the failure class.
2. Was the underlying store healthy? (Vector store, BM25 index, reranker API.)
3. Were rate limits hit?
4. Were credentials expired?

**Common causes.**
- Vector store overloaded.
- Provider API rate-limited.
- Credentials expired.
- Network connectivity issue.

**Remediation.** Fall back per [fallback-patterns.md](../reliability-engineering/fallback-patterns.md). Address the underlying cause.

---

## 4. Generation-stage failures

Failures where retrieval was correct but the model produced a wrong answer.

### 4.1 The misused-context failure

**Symptom.** The model had the right retrieved context but produced an answer that didn't reflect it.

**Diagnostic.**
1. Trace: confirm retrieval returned the right content (per section 3).
2. Read the actual prompt that was sent to the model (per the prompt assembly).
3. Was the retrieved content in the prompt? (Verify chunks were formatted into the prompt.)
4. Was the prompt too long? (Did the model lose track due to context length?)
5. Was the model's instruction clear about how to use retrieved content?

**Common causes.**
- Retrieved chunks were formatted in a way the model didn't recognize.
- The prompt's instructions about using context were unclear.
- Context-window pressure caused the model to focus on later content (lost-in-the-middle).
- The retrieved chunk had relevant info but in a section the model skipped.

**Remediation.** Prompt engineering (clearer instructions; better chunk formatting); reduce context to the most-relevant chunks; eval the change.

### 4.2 The hallucinated-claim failure

**Symptom.** The model made a claim that isn't supported by the retrieved content.

**Diagnostic.**
1. Trace: identify the specific claim.
2. Read the retrieved chunks; verify the claim isn't supported.
3. Is the claim possibly from the model's training data rather than retrieval?
4. Did the model's instructions explicitly say "only use retrieved content"?

**Common causes.**
- The model defaulted to training-data knowledge when retrieval was insufficient.
- The model conflated multiple retrieved chunks into a synthesized claim that no single chunk supports.
- The prompt didn't strongly constrain the model to retrieved content.
- The retrieved context was thin (one chunk; the model filled in around it).

**Remediation.** Tighten the system prompt's grounding instructions; ensure citation requirement; add faithfulness eval cases.

### 4.3 The hallucinated-citation failure

**Symptom.** The model cited a source, but the source either doesn't exist or doesn't contain the cited claim.

**Diagnostic.**
1. Trace: identify the cited source.
2. Verify the source ID exists in the retrieval results.
3. Read the source's content; verify the citation accuracy.

**Common causes.**
- The model invented a citation that sounds plausible.
- The model cited the wrong chunk (used the first one in context regardless of relevance).
- The model cited based on training data not retrieval.

**Remediation.** Citation validation per [eval-of-rag.md](../eval-engineering/eval-of-rag.md) section 5; reject hallucinated citations; add regression case.

### 4.4 The off-topic-response failure

**Symptom.** The model's response is on a different topic than the user asked.

**Diagnostic.**
1. Trace: read the question and the response.
2. Did the retrieval return off-topic chunks? (Wrong-retrieval per section 3.3.)
3. Did the model pivot from the user's question to a related topic?
4. Was the prompt's instruction about staying on topic clear?

**Common causes.**
- Wrong retrieval (retrieval issue, not generation).
- Model focused on a tangent in the retrieved content.
- The user's question was ambiguous; the model chose a different interpretation than the user intended.

**Remediation.** Investigate retrieval first (per section 3); then prompt engineering to constrain topic adherence.

### 4.5 The wrong-format response failure

**Symptom.** The response doesn't follow the expected format (missing structure, wrong fields, malformed JSON).

**Diagnostic.**
1. Trace: read the response.
2. Was the prompt's format instruction clear?
3. Was structured output enforced (JSON Schema, tool-calling)?
4. Did the model encounter unfamiliar content that broke its formatting?

**Common causes.**
- Prompt's format instructions were vague.
- Model wasn't given structured-output constraints.
- Edge-case input that the format doesn't accommodate.

**Remediation.** Strengthen format instructions; use structured output (JSON Schema, tool-calling per [structured-output-engineering.md](../prompt-engineering/) coming); validate and repair.

### 4.6 The over-refusal failure

**Symptom.** The model refused to answer when it should have answered.

**Diagnostic.**
1. Trace: confirm the refusal.
2. Was relevant content in retrieval?
3. Was the refusal triggered by content-policy or safety rules?
4. Was the refusal triggered by the model's overcautious interpretation?

**Common causes.**
- The retrieved content contained sensitive terms; the model treated the whole interaction as sensitive.
- The system prompt's refusal criteria were too broad.
- Model's training biases toward refusal on certain topic classes.

**Remediation.** Tune the system prompt's refusal criteria; add eval cases for false-refusal; potentially change tier.

### 4.7 The under-refusal failure

**Symptom.** The model answered when it should have refused (e.g., produced clinical advice without sufficient retrieval support).

**Diagnostic.**
1. Trace: confirm the response.
2. Was retrieval thin (no good support for the claim)?
3. Should the model have refused with "I don't have information on that"?

**Common causes.**
- System prompt didn't have strong-enough "refuse on uncertainty" instructions.
- Retrieval returned chunks the model deemed sufficient but were actually thin.

**Remediation.** Strengthen the refusal-when-uncertain instructions; add eval cases for empty-retrieval handling per [eval-of-rag.md](../eval-engineering/eval-of-rag.md) section 6.

---

## 5. Corpus-stage failures

Failures originating in the corpus itself.

### 5.1 The missing-document failure

**Symptom.** The relevant document isn't in the corpus.

**Diagnostic.**
1. Verify the document doesn't exist in the corpus.
2. Was the document expected to be there? (Check the corpus inventory; check upstream availability.)
3. Was the document in the corpus and removed? (Check ingestion / deletion history.)
4. Is the document in upstream but not yet ingested? (Check ingestion lag.)

**Common causes.**
- Document was never ingested (upstream wasn't connected; document wasn't in the ingestion scope).
- Document was quarantined per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) section 7.2.
- Document was deleted (intentionally or accidentally).
- Ingestion is behind; the document will appear after the next run.

**Remediation.** Ingest the document; investigate the gap; add freshness monitoring.

### 5.2 The corrupted-document failure

**Symptom.** The document is in the corpus but its content is garbled.

**Diagnostic.**
1. Read the chunk content; verify it's garbled.
2. Read the source document; verify the original isn't garbled.
3. Was the format conversion lossy? (Per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) section 4.)
4. Was the chunk extracted incorrectly?

**Common causes.**
- Format conversion failure (PDF with embedded images; OCR error).
- Encoding issue (mojibake; wrong character set).
- HTML parsing error.

**Remediation.** Fix the converter; re-ingest affected documents.

### 5.3 The stale-document failure

**Symptom.** The document in the corpus is older than the upstream version.

**Diagnostic.**
1. Check the document's `last_modified` in the corpus.
2. Check the source's current state.
3. Was there a recent update upstream?
4. Why didn't the ingestion catch it?

**Common causes.**
- Connector's change-detection missed the update.
- Ingestion failed silently and didn't retry.
- Hash collision (the update content happened to produce the same hash).

**Remediation.** Trigger re-ingestion; fix the change-detection.

### 5.4 The duplicated-document failure

**Symptom.** The same document appears multiple times in the corpus, possibly with different IDs.

**Diagnostic.**
1. Search the corpus for the duplicate.
2. Were they ingested from different sources?
3. Did deduplication fail?

**Common causes.**
- Same document appears in multiple upstream sources without coordinated IDs.
- Deduplication is broken or absent.
- Content hash differs slightly (whitespace, formatting) so deduplication doesn't match.

**Remediation.** Improve deduplication; mark duplicates as such.

---

## 6. Configuration-stage failures

Failures from wrong configuration.

### 6.1 The wrong-model-version failure

**Symptom.** Behavior changed; new investigation shows a different model version than expected.

**Diagnostic.**
1. Trace: confirm `ai.llm.model_version`.
2. Check the release manifest per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).
3. Did an alias resolve to a new version?

**Common causes.**
- Model alias resolved to a new version.
- Release manifest was updated without eval validation.
- Manual override left in place.

**Remediation.** Pin the version; re-deploy; eval against the correct version.

### 6.2 The wrong-prompt-version failure

**Symptom.** Behavior changed; investigation shows a different prompt version than expected.

**Diagnostic.**
1. Trace: confirm `ai.llm.prompt.version`.
2. Check the release manifest.
3. When did the prompt change?

**Common causes.**
- Prompt change shipped without eval validation.
- Wrong version pinned in the manifest.

**Remediation.** Roll back to the previous prompt; investigate the change.

### 6.3 The wrong-corpus-version failure

**Symptom.** Retrieval against a different corpus version than expected.

**Diagnostic.**
1. Trace: confirm `ai.retrieval.corpus_version`.
2. Check the corpus snapshot history.

**Common causes.**
- Reading from a stale corpus replica.
- Corpus version pin missing in the release manifest.

**Remediation.** Verify corpus pin; re-deploy.

### 6.4 The wrong-filter failure

**Symptom.** Retrieval returns results from a different scope than expected.

**Diagnostic.**
1. Trace: confirm `ai.retrieval.tenant_filter` and other filter attributes.
2. Compare against expected.

**Common causes.**
- Filter logic bug.
- Filter not propagated through the pipeline.

**Remediation.** Fix the filter; canary test; eval.

---

## 7. The diagnostic workflow framework

The general framework that diagnostic walkthroughs follow.

### 7.1 The intake

When an incident is reported:

1. Capture: user description, interaction ID (or session ID), time of occurrence, expected vs actual behavior.
2. Confirm reproducibility: can the issue be reproduced from the trace?

### 7.2 The trace pull

Pull the trace for the affected interaction. The trace is the primary evidence.

### 7.3 The top-down read

Read the trace top-down:

1. **Top-level interaction span.** Outcome; total cost; total latency.
2. **Per-turn spans (for agents).** Which turn failed?
3. **LLM call spans.** Which model; which prompt version; finish reason.
4. **Retrieval spans.** What was queried; what was returned; what scores.
5. **Tool call spans.** Which tools called; authorization decisions.

The read identifies the stage where things diverged from expected.

### 7.4 The hypothesis formation

From the trace, form a hypothesis: "Stage X failed because Y."

Cross-reference: per the failure taxonomy (sections 3-6), which class does this match?

### 7.5 The deep-dive

For the hypothesized cause, do the deep-dive diagnostics per the relevant section.

### 7.6 The conclusion

The diagnostic ends with:
- Root cause identified.
- Remediation chosen.
- Regression case planned (per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md)).

### 7.7 The post-incident actions

- Apply the fix.
- Add the regression case.
- Update the runbook if the workflow surfaced a gap.
- If observability was inadequate, add instrumentation.
- If the failure pattern was new, add it to the failure-mode catalog.

### 7.8 The diagnostic latency target

For Sev-1 / Sev-2 incidents: time-to-diagnosis < 1 hour. The workflow supports this.

For lower-severity issues: diagnosis within the next business day.

---

## 8. Remediation patterns

The common remediation patterns per failure class.

### 8.1 Retrieval remediation

- Tune retrieval parameters (per [retrieval-engineering.md](./retrieval-engineering.md)).
- Improve chunking (per [chunking-engineering.md](./chunking-engineering.md)).
- Add query rewriting for the affected query class (per [query-rewriting.md](./query-rewriting.md)).
- Adjust rerank threshold (per [reranking-engineering.md](./reranking-engineering.md)).

### 8.2 Generation remediation

- Strengthen prompt instructions.
- Add citation validation per [eval-of-rag.md](../eval-engineering/eval-of-rag.md) section 5.
- Add faithfulness eval cases.
- Consider a different model tier (per [tier-routing-for-cost.md](../cost-and-finops/tier-routing-for-cost.md)).

### 8.3 Corpus remediation

- Trigger re-ingestion of the affected source per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md).
- Fix the converter / chunker.
- Update freshness SLO monitoring per [data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md).

### 8.4 Configuration remediation

- Roll back the wrong configuration (model version, prompt version).
- Update pinning per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).
- Re-deploy with the correct configuration.

### 8.5 Each remediation includes a regression case

Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md), every fix produces a regression case. The case prevents recurrence.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The 2026-Q2 cardiology incident

The cardiology guidelines incident referenced in [chunking-engineering.md](./chunking-engineering.md) section 9.5 and [retrieval-observability.md](./retrieval-observability.md) section 9.4. Walking through the diagnostic:

**Intake.** Online judge SLI alert: cardiology recall dropped from 89% to 78%. Multiple failed interactions observed.

**Trace pull.** Pulled the most recent failed interactions' traces.

**Top-down read.**
- Top-level: agent completed normally, but generation was wrong.
- Retrieval span: expected sources (cardiology guideline chunks) weren't in returned doc_ids.
- Per-retriever sub-spans: BM25 retrieved them at rank 28-35; vector at rank 22-29. Both retrieved them but past top-20.
- Rerank: kept them in top-10? No — they ranked past 10 after rerank too.

**Hypothesis.** Retrieval-stage failure: missed-expected-source per section 3.2. The expected source is in the corpus but not in top-K.

**Deep-dive.** Investigated chunks themselves; found they had fewer tokens than expected. The chunker had produced smaller-than-normal chunks for the cardiology updates.

**Root cause.** AHA released updated cardiology guidelines with SVG figures; the converter's text extraction failed on figure-heavy sections; chunks were produced from the truncated text.

**Remediation.**
1. Converter updated to handle SVG figures.
2. Affected documents re-ingested.
3. Recall restored to 89%.
4. Regression case added: a cardiology question that specifically tests SVG-heavy guideline retrieval.
5. Runbook updated: add a "check for SVG / figure handling" step in the converter section.

Total time: ~3 hours from alert to fix-deployed. The runbook accelerated the diagnosis.

### 9.2 The 2026-Q2 dosing incident

Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) section 9.2 (BUG-2026-0089, the warfarin dosing issue).

**Intake.** Clinician reported the system gave standard warfarin dose for an elderly patient with renal impairment.

**Trace pull.** Pulled the affected interaction.

**Top-down read.**
- Retrieval span: returned the standard-warfarin-dosing chunk; did not return the elderly-renal-considerations chunk.
- Per-retriever: BM25 had elderly-renal-considerations at rank 15; vector at rank 12.
- The elderly-renal-considerations chunk made the top-50 but not the top-10.

**Hypothesis.** Generation-stage failure (the elderly-renal-considerations chunk was retrievable but not retrieved at top-K).

Wait — let me re-read. Actually the chunk was retrieved at rank 12 (vector) and 15 (BM25); after RRF and rerank, it didn't make top-10. So the retrieval *almost* found it but it was filtered out.

Actually, on closer trace inspection: the chunk did make top-10 but the supervisor prompt didn't have instructions emphasizing elderly-patient context, so the clinical-knowledge worker (which received top-10 in its context) didn't focus on the elderly-considerations chunk.

**Root cause** (revised): The supervisor prompt didn't sufficiently emphasize elderly-patient context for clinical workers.

**Remediation.**
1. Supervisor prompt updated with stronger elderly-patient handling.
2. Regression case added: the elderly-renal-warfarin question.
3. Eval validated.
4. Deployed.

The diagnostic distinguished retrieval (the chunk was retrieved) from generation (the model didn't emphasize it) — exactly the distinction per [eval-of-rag.md](../eval-engineering/eval-of-rag.md) section 4.

### 9.3 The runbook structure

Each diagnostic workflow per sections 3-6 is a runbook page. The structure:

- Trigger.
- Diagnostic steps (sequential).
- Common causes.
- Remediation pattern.
- Escalation criteria.

Runbooks are rehearsed quarterly per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md).

### 9.4 The failure-mode catalog

Meridian's catalog has 47 documented failure modes as of 2026-Q2. The catalog grew from the original ~15 to 47 as the team encountered and documented new patterns.

The catalog is the team's accumulated knowledge of how this specific system fails. New on-call engineers consult the catalog before guessing.

### 9.5 The platform discipline

- Trace is the primary diagnostic evidence.
- Runbook per failure-mode class.
- Catalog updated as new patterns surface.
- Every fix produces a regression case.
- Post-incident reviews include observability gap analysis.

---

## 10. Anti-patterns

### 10.1 "Diagnose by re-running"

On-call re-runs the question to see what happens; gets a different result (model nondeterminism); investigation drifts.

**Corrective.** Trace is the primary evidence per section 7. Re-running is secondary.

### 10.2 "No failure catalog"

Each investigation starts from scratch; the team doesn't accumulate knowledge.

**Corrective.** Maintain the failure-mode catalog per section 9.4.

### 10.3 "Symptom-to-cause guessing"

Investigation jumps from symptom to assumed cause; deep-dive misses the actual issue.

**Corrective.** Top-down trace read per section 7.3; hypothesis before deep-dive.

### 10.4 "Fix without regression case"

Bug is fixed; no regression case added; the bug recurs months later.

**Corrective.** Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md), regression case is part of the fix.

### 10.5 "Runbooks absent"

Each on-call works out the diagnostic alone; no documented workflows; investigation latency is high.

**Corrective.** Runbooks per section 4 / 9.3.

### 10.6 "Failure mode not characterized"

A bug is fixed but the failure mode isn't named or catalogued; similar bugs in the future aren't recognized.

**Corrective.** After each fix, characterize the failure mode; add to catalog if new.

### 10.7 "Observability gap unaddressed"

Investigation reveals the trace didn't have enough information; the team works around it; the gap persists for the next investigation.

**Corrective.** Post-incident review identifies observability gaps; instrumentation added.

### 10.8 "Same-stage assumption"

Investigation assumes the bug is in retrieval (since the team's last few bugs were in retrieval); deep-dive in retrieval; the actual issue is in generation.

**Corrective.** Stage-by-stage analysis per section 7.3 without assuming.

---

## 11. Findings (sprint-assignable)

### RAGDBG-001 — Severity: Critical
**Finding.** No failure-mode catalog; each investigation starts from scratch.
**Recommendation.** Initialize the catalog per section 2; populate from recent incident history.
**Owner.** ai-platform-eng team lead, sprint N+1.

### RAGDBG-002 — Severity: Critical
**Finding.** Runbooks for failure modes do not exist; on-call diagnosis is ad-hoc.
**Recommendation.** Per-class runbooks per sections 3-6; rehearsed quarterly.
**Owner.** ai-platform-eng + sre, sprint N+1.

### RAGDBG-003 — Severity: Critical
**Finding.** Investigations re-run queries instead of reading existing traces.
**Recommendation.** Trace-as-evidence discipline per section 7.3.
**Owner.** ai-platform-eng team lead, sprint N+1.

### RAGDBG-004 — Severity: High
**Finding.** Regression cases not added for diagnosed bugs.
**Recommendation.** Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md).
**Owner.** ai-platform-eng, sprint N+2.

### RAGDBG-005 — Severity: High
**Finding.** Stage-by-stage diagnostic analysis not documented; investigations jump to assumed causes.
**Recommendation.** Top-down trace read per section 7.3; document the framework.
**Owner.** ai-platform-eng team lead, sprint N+2.

### RAGDBG-006 — Severity: High
**Finding.** Post-incident reviews don't include observability gap analysis.
**Recommendation.** Add to post-mortem template; surface gaps; deploy instrumentation.
**Owner.** ai-platform-eng + sre, sprint N+2.

### RAGDBG-007 — Severity: High
**Finding.** Time-to-diagnosis exceeds 1 hour for Sev-1 / Sev-2; runbooks don't accelerate.
**Recommendation.** Runbook quality improvement per section 4.6; rehearse.
**Owner.** ai-platform-eng + sre, sprint N+3.

### RAGDBG-008 — Severity: Medium
**Finding.** Same-stage assumption bias; team's recent bugs influence next investigation.
**Recommendation.** Discipline per section 7.3; checklist forces stage-by-stage.
**Owner.** ai-platform-eng team lead, sprint N+3.

### RAGDBG-009 — Severity: Medium
**Finding.** New failure modes not catalogued; team rediscovers patterns.
**Recommendation.** Catalog update is part of every fix per section 9.4.
**Owner.** ai-platform-eng, sprint N+3.

### RAGDBG-010 — Severity: Medium
**Finding.** Retrieval-stage diagnostic conflates wrong-retrieval with missed-expected-source.
**Recommendation.** Distinguish per sections 3.2 vs 3.3.
**Owner.** ai-platform-eng, sprint N+3.

### RAGDBG-011 — Severity: Medium
**Finding.** Generation-stage diagnostic doesn't check format-validation; format issues undetected.
**Recommendation.** Add format-validation diagnostic per section 4.5.
**Owner.** ai-platform-eng, sprint N+3.

### RAGDBG-012 — Severity: Medium
**Finding.** Corpus-stage diagnostic doesn't check ingestion lag; stale-document failures undiagnosed.
**Recommendation.** Add ingestion-lag check per section 5.3.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### RAGDBG-013 — Severity: Medium
**Finding.** Configuration-stage diagnostic doesn't have a clear path to release-manifest verification.
**Recommendation.** Per section 6; release manifest as primary configuration evidence.
**Owner.** ai-platform-eng + sre, sprint N+4.

### RAGDBG-014 — Severity: Medium
**Finding.** Diagnostic latency target not tracked; team has no SLO for time-to-diagnosis.
**Recommendation.** Track per section 7.8; report monthly.
**Owner.** ai-platform-eng + sre, sprint N+4.

### RAGDBG-015 — Severity: Medium
**Finding.** Scope-violation diagnostic is treated as generic incident; not Sev-1.
**Recommendation.** Scope violation is Sev-1 per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md).
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### RAGDBG-016 — Severity: Low
**Finding.** Runbook documentation thin in places; new on-call cannot follow easily.
**Recommendation.** Runbook clarity review; rehearse with new on-call.
**Owner.** ai-platform-eng team lead, sprint N+5.

### RAGDBG-017 — Severity: Low
**Finding.** Failure-mode catalog not indexed; engineers can't quickly find prior cases.
**Recommendation.** Searchable catalog; tag-based browsing.
**Owner.** ai-platform-eng, sprint N+5.

### RAGDBG-018 — Severity: Low
**Finding.** Production replay not used for fix validation; investigations don't verify before deploy.
**Recommendation.** Replay per [retrieval-observability.md](./retrieval-observability.md) section 8.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team without diagnostic discipline:

- [ ] **Sprint 0 — design.** Failure-mode taxonomy per section 2; identify the stages relevant to the workload.
- [ ] **Sprint 1 — first runbooks.** Three or four runbooks covering the most common failure classes.
- [ ] **Sprint 1 — diagnostic framework.** Top-down trace read per section 7.3 documented.
- [ ] **Sprint 2 — failure-mode catalog.** Initialize from recent incidents; format consistent.
- [ ] **Sprint 2 — regression discipline.** Every fix produces a regression case per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md).
- [ ] **Sprint 3 — additional runbooks.** Cover remaining failure classes; rehearse.
- [ ] **Sprint 3 — post-incident reviews.** Observability gap analysis integrated.
- [ ] **Sprint 4 — observability fills.** Address gaps surfaced by post-incident reviews.
- [ ] **Sprint 4 — diagnostic latency.** Track time-to-diagnosis; report.
- [ ] **Sprint 5 — refinement.** Quarterly review of catalog, runbooks, latency.
- [ ] **Ongoing — discipline.** Trace-first investigations; catalog growth; runbook rehearsal.

A team that completes this sequence turns investigations from hours-of-guessing into 30-minute reads-of-the-trace. A team that skips the discipline pays in repeated bugs and slow diagnosis.

---

## 13. References

- This repo: [rag-engineering/ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) — corpus-stage failures.
- This repo: [rag-engineering/chunking-engineering.md](./chunking-engineering.md) — chunking failures.
- This repo: [rag-engineering/embedding-pipeline-engineering.md](./embedding-pipeline-engineering.md) — embedding-stage failures.
- This repo: [rag-engineering/retrieval-engineering.md](./retrieval-engineering.md) — retrieval-stage failures.
- This repo: [rag-engineering/reranking-engineering.md](./reranking-engineering.md) — rerank-stage failures.
- This repo: [rag-engineering/query-rewriting.md](./query-rewriting.md) — query-rewriting failures.
- This repo: [rag-engineering/retrieval-observability.md](./retrieval-observability.md) — observability that supports diagnosis.
- This repo: [rag-engineering/rag-eval-integration.md](./rag-eval-integration.md) — eval discipline.
- This repo: [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — fix → regression-case workflow.
- This repo: [eval-engineering/eval-of-rag.md](../eval-engineering/eval-of-rag.md) — eval discipline for diagnosis.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — trace evidence.
- This repo: [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — agent trace evidence.
- This repo: [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alert hierarchy.
- This repo: [reliability-engineering/fallback-patterns.md](../reliability-engineering/fallback-patterns.md) — fallback patterns for retrieval failures.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md) — scope violation incident response.
