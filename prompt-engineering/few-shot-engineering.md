# Few-Shot Engineering

> **Audience.** Engineers and prompt authors deciding whether to add few-shot examples to a prompt, how many, which ones, in what order. Tech leads weighing few-shot vs fine-tuning. Anyone whose prompts have grown from one paragraph + zero examples to one paragraph + 25 examples without explicit examination. **Scope.** The engineering practice of few-shot example use — when it earns its tokens, example selection (curated vs retrieved), ordering and presentation, the diminishing-returns curve, and the migration to fine-tuning when the example budget grows past a threshold. **Worked client.** Meridian Health.

---

## 1. Why this document exists

Few-shot examples are tokens. Every example in the prompt is paid for at input rates on every call. A prompt with 20 examples (each ~150 tokens) pays 3,000 input tokens per call for the examples alone — at frontier model rates, that's $0.009 per call before the actual user input is added. Across high-volume features, the examples become a significant fraction of the cost envelope.

Few-shot examples can also be the difference between a feature that works and one that doesn't. A model that consistently mis-formats output on a structured task gets fixed by 2–5 examples of the expected format. A classifier that hallucinates labels gets corrected by examples of the correct labels. A reasoning task that veers off-pattern gets anchored by examples of the correct pattern.

The engineering question is not "should we use few-shot?" but "what is the smallest example set that produces the quality we need?" Most production few-shot use is over-budgeted: the team added examples until the model worked, then never revisited. The original 12-example set has grown to 23 examples through cumulative additions, of which 8 are redundant, 3 actively hurt on some inputs, and 2 are bugs (typos in the example output that the model now learns to replicate).

The discipline this document covers: deliberate selection, measured impact, regular pruning. Few-shot examples are engineered artefacts; they have the same versioning, ownership, and eval treatment as the surrounding prompt.

This document is opinionated about four things:

1. **Few-shot earns its tokens or doesn't ship.** Each example is justified by an eval delta. Examples added without eval support are pruned at the next audit.
2. **Curation beats retrieval for most use cases.** Dynamic example retrieval is more complex; for most tasks, a small curated set outperforms a large retrieved set per unit cost.
3. **There is a diminishing-returns curve.** Quality gain per added example flattens fast — typically by example 5–8 for most tasks. Adding examples past the curve's knee pays cost for negligible gain.
4. **Beyond ~20 examples, examine fine-tuning.** The crossover point where fine-tuning produces better quality per dollar is approximately 20+ curated examples plus stable use case. Below 20, few-shot dominates; above, fine-tune becomes competitive.

Structure: (2) when few-shot earns its tokens; (3) example selection — curated vs retrieved; (4) example ordering and presentation; (5) the diminishing-returns curve; (6) the few-shot-to-fine-tune transition; (7) example set management at scale; (8) eval and observability for few-shot; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. When few-shot earns its tokens

The decision is per-task, not per-prompt-author preference.

### 2.1 Cases where few-shot reliably helps

- **Format anchoring.** The output's shape is specific (JSON with particular fields, markdown with particular sections, structured citation format). 2–5 examples anchor the format.
- **Style anchoring.** The output's tone or register is specific (medical professional vs casual, terse vs verbose, formal vs informal). Examples demonstrate the target.
- **Classification with rare or fuzzy classes.** The classes are not obvious from name; examples disambiguate. Particularly useful for domain-specific classes.
- **Edge case calibration.** Specific edge cases that the model handles poorly without seeing the right pattern (e.g., empty input handling, ambiguity resolution).
- **Domain-specific reasoning patterns.** The reasoning style the team wants (e.g., "consider X, then Y, then Z") is anchored by examples showing it.

### 2.2 Cases where few-shot doesn't help much

- **Generic tasks the model already does well.** "Summarise this article" doesn't need examples; the model knows what a summary is.
- **Tasks where the instruction is unambiguous and the model follows it.** "Translate this to Spanish" needs no examples.
- **Tasks where the variation across inputs is too high for a few examples to be representative.** Examples bias the model toward the example pattern; if inputs are diverse, the bias hurts.
- **Tasks where structured outputs + schema enforcement (per [structured-output-engineering.md](./structured-output-engineering.md)) accomplish the same thing more reliably.**

### 2.3 The "do I even need examples?" test

Before adding examples:

1. Write the prompt without examples.
2. Run on 20 representative inputs.
3. Note the failure modes.
4. For each failure mode, ask: is this a "the model doesn't know the format / style / pattern" failure (few-shot helps), or a "the model can't do this task" failure (better prompt / better model needed)?
5. Add examples only for the failure modes few-shot addresses.

The test prevents reflexive example-adding. Many tasks pass step 2 without needing few-shot at all.

### 2.4 The cost-benefit framing

Each example costs input tokens × call volume. Each example provides quality gain (measured on eval). The decision:

```
add example if: quality_gain × value_per_quality_point > example_token_cost × call_volume × rate
```

In practice, teams use simpler heuristics: "if the eval shows >1% improvement, the example is worth its cost on a high-volume feature." For low-volume features, the bar is lower (tokens are cheap relative to engineering effort).

### 2.5 Few-shot interacts with other techniques

Few-shot composes with:

- **Structured output enforcement.** Schema constrains the shape; few-shot anchors the style within the shape.
- **System prompt anchoring.** The system prompt sets context; few-shot demonstrates application.
- **Chain-of-thought prompting.** Examples demonstrate the reasoning pattern, not just the input-output.

It substitutes for:

- **Detailed instructions.** Examples are often more concise than verbose instructions describing the same thing.
- **Fine-tuning for small-scale tasks.** When the task is bounded and the example set is small, few-shot delivers similar quality at lower engineering cost.

### 2.6 The model-tier dimension

Smaller models benefit more from few-shot than larger ones. A frontier model often handles a task with zero examples; a smaller model needs 3–8 examples to match. When tier-routing (per [tier-routing-for-cost.md](../cost-and-finops/tier-routing-for-cost.md)), the few-shot budget may need to differ per tier — more examples for the cheap model, fewer (or none) for the expensive one.

---

## 3. Example selection — curated vs retrieved

Two paradigms for choosing which examples to include. Each fits different cases.

### 3.1 Curated (static) examples

A small, hand-picked set of examples lives in the prompt. Every call uses the same examples regardless of input.

**Pros.**

- Predictable token cost (always the same).
- Simple to engineer and test (no retrieval pipeline).
- Stable behaviour (no risk of example-set drift causing quality variance).
- Easy to version (the prompt's version captures the examples).

**Cons.**

- The same examples may not be optimal for all inputs.
- Diverse input distributions are imperfectly served.

**When right.** Tasks with a narrow input shape; format anchoring; small example budget (< 10 examples).

### 3.2 Retrieved (dynamic) examples

For each call, examples are retrieved from a larger pool based on similarity to the current input.

**Pros.**

- Examples are relevant to the specific input.
- Larger effective example pool than a curated set can hold.
- Adapts to input variation.

**Cons.**

- Retrieval infrastructure (vector store, embeddings, similarity search).
- Token cost varies per call (sometimes more, sometimes less than curated).
- Retrieval quality is itself a failure surface.
- Harder to version (the example pool changes; the retrieval ranks; the resulting prompt is not deterministic).
- Risk of "retrieval echo chamber" — similar inputs retrieve similar examples, biasing the model toward narrow patterns.

**When right.** Highly diverse input distribution; large example pool available; the variance in optimal example differs significantly across inputs.

### 3.3 The decision

| Criterion | Curated | Retrieved |
| --- | --- | --- |
| Input diversity | Low | High |
| Example pool size needed | < 10 | 50+ |
| Predictable token cost matters | Yes | No |
| Eval and versioning simplicity | Important | Manageable |
| Engineering complexity acceptable | Low only | Higher |
| The "right" example varies per input | No | Yes |

Default to curated. Move to retrieved only when curation provably underserves and the engineering investment is justified.

### 3.4 The hybrid pattern

A small curated set (3–5 examples for format / style anchoring) + 2–4 retrieved examples (for input-specific guidance). Captures the predictability of curation and the specificity of retrieval.

The hybrid is the most common production pattern. The curated examples set the baseline; the retrieved examples address the specific case.

### 3.5 Retrieved example engineering

When using retrieved examples:

- **Pool curation.** The pool itself is curated; not every historical input is a good example. Verified examples only.
- **Diversity in retrieval.** Pure top-N-similar can over-retrieve from one cluster; mix similarity with diversity (MMR or similar).
- **Negative-example exclusion.** Don't retrieve examples the team has flagged as misleading or outdated.
- **Token budget enforcement.** Cap total retrieved tokens; if top-3-similar exceed the budget, retrieve fewer or summarise.
- **Per-call observability.** Trace logs which examples were retrieved for the call; supports debugging "why did the agent give that answer."

### 3.6 The "examples drift" failure mode

Retrieved-example systems drift as the pool changes. New examples are added; old ones become less representative; quality changes silently. Discipline:

- Pool changes go through review (like prompt changes).
- Eval rerun after pool updates.
- Continuous eval monitors quality stability.

The retrieval system inherits the prompt's eval discipline plus the pool's curation discipline.

---

## 4. Example ordering and presentation

How examples are formatted and ordered affects model behaviour.

### 4.1 Ordering effects

The model attends more to recent context. Examples toward the end of the example block influence the model more than those at the start. This means:

- **Strongest / most representative examples last.** The model's most-recent-attention emphasis means the last 1–2 examples have outsized influence.
- **Hard cases late.** The model has more context (the earlier examples) when it sees the hard one; it has the pattern established.
- **Avoid the same input format throughout.** Variation in input shape (when realistic) demonstrates the pattern's flexibility.

The ordering matters less for some tasks (clear format anchoring) and more for others (subtle reasoning patterns).

### 4.2 Diversity in the example set

Examples that cover different shapes of input outperform examples that repeat similar shapes. A 5-example set with 5 different input patterns is more useful than a 10-example set with 2 patterns each represented 5 times.

For classification: include examples of each class. For tasks with edge cases: include the edge cases.

### 4.3 Negative examples

For tasks where the model has tendencies the team wants to correct, negative examples ("this is what NOT to do") help. Pattern:

```
Example 1 (correct):
Input: [...]
Output: [...]

Example 2 (correct):
Input: [...]
Output: [...]

Example 3 (incorrect — do not do this):
Input: [...]
Output: [WRONG ANSWER]
Reason: [why this is wrong]

Example 4 (correct):
Input: [...]
Output: [...]
```

Use sparingly. Negative examples can confuse some models; test against the eval set.

### 4.4 Example formatting

Consistency matters. The format the examples use is the format the model produces. If the example outputs are JSON with specific field ordering, the model tends to produce JSON with that ordering. If the examples vary in formatting, the model produces inconsistent output.

Discipline:

- Example outputs match the exact format expected.
- Field ordering, whitespace, quoting — all consistent.
- The example's input format matches the production input format.

### 4.5 The chain-of-thought variant

When the task benefits from step-by-step reasoning, examples show the reasoning:

```
Example:
Input: [...]
Reasoning: First, I [step]. Then, I [step]. Finally, I conclude [step].
Output: [...]
```

The model learns both the reasoning pattern and the output. Often more useful than the input-output pair alone for reasoning tasks. Token cost is higher; quality gain often justifies it.

### 4.6 Example labelling

Label each example clearly:

```
Example 1:
Input: ...
Output: ...

Example 2:
Input: ...
Output: ...
```

The labels make the structure explicit; the model treats each example as a unit. Without labels, examples can blur into prose; the model may misinterpret.

### 4.7 The "show, don't tell" tradeoff

Instructions are tokens too. The trade-off:

- "Output should be valid JSON with fields name, age, location" (verbose instruction).
- One example showing the JSON structure (shorter; more reliable for some tasks).

For format / style, examples often beat instructions per token. For complex multi-condition logic, explicit instructions plus 1–2 disambiguating examples is often best.

---

## 5. The diminishing returns curve

Quality gain per added example flattens fast. Engineering accordingly.

### 5.1 The typical shape

| Examples | Quality (% correct) | Per-example marginal gain |
| --- | --- | --- |
| 0 | 71% | (baseline) |
| 1 | 78% | +7% |
| 2 | 83% | +5% |
| 3 | 86% | +3% |
| 5 | 89% | +1.5% / example |
| 8 | 91% | +0.7% / example |
| 12 | 92% | +0.25% / example |
| 20 | 92.5% | +0.0625% / example |

Numbers are illustrative; the shape is typical. Most of the quality gain is in the first 3–5 examples; subsequent examples add small marginal gains; past ~10 examples, the marginal gain is often within eval noise.

### 5.2 The knee of the curve

The "knee" — where marginal gain drops below cost-justification — is usually 5–8 examples for most tasks. Beyond the knee, additional examples pay token cost for negligible quality gain.

Some tasks have knees later (complex multi-class classification might have a knee at 15+ examples). Some have knees earlier (simple format anchoring at 2–3 examples).

### 5.3 Measuring your task's curve

The team measures by adding examples one at a time and re-running eval:

1. Baseline (zero examples).
2. Add example 1; eval.
3. Add example 2; eval.
4. Continue until marginal gain drops below threshold (e.g., < 0.5% per added example for 2 consecutive examples).
5. Lock the set; tag the version.

The exercise takes hours, not days. Most prompts in production are using more examples than their curves justify.

### 5.4 The audit cadence

Quarterly, the team re-runs the curve analysis on the current prompt's example set. Examples whose individual contribution drops below threshold are candidates for removal.

A typical audit removes 1–3 examples from a mature prompt. The removals reduce cost without measurable quality loss.

### 5.5 Why the curve flattens

The early examples teach the pattern. Subsequent examples are mostly redundant — they reinforce what the first few established. Past a point, each new example is "more of the same"; the model has already learned.

This is task-dependent. Tasks with high variance across input patterns benefit from more examples (each example teaches a different pattern). Tasks with narrow input patterns saturate fast.

### 5.6 The "more is better" heuristic is wrong

Engineers' intuition is often "more examples → better." On the curve, the truth is "more examples → diminishing returns; past the knee, increasing cost without quality gain." Resist the heuristic; measure.

---

## 6. The few-shot-to-fine-tune transition

When few-shot's curve flattens and the task is high-volume, fine-tuning becomes worth considering.

### 6.1 The crossover criteria

Fine-tuning is worth examining when:

- **Few-shot example count exceeds 20.** The token cost is significant; fine-tune absorbs the pattern into the weights at zero per-call token cost.
- **Use case is stable.** Frequent prompt changes invalidate the fine-tune; fine-tuning is right for settled use cases.
- **Quality gap remains.** Few-shot has plateaued below the target; fine-tune may break through.
- **Volume is high.** The per-call token savings × volume × rate justifies the fine-tune investment.

If three of four apply, fine-tune is a serious option.

### 6.2 The cost framing

Per-call cost comparison:

- **Few-shot at 20 examples (~3000 tokens):** $0.009 per call at sonnet rates.
- **Fine-tuned model:** input rate × actual user input only (no example tokens). At 500 user-input tokens, $0.0015 per call.
- **Per-call savings:** $0.0075. At 100k calls/day, $750/day, $22,500/month.

The fine-tune investment ($1,000–$10,000 typically) pays back in weeks to months on high-volume features.

### 6.3 The fine-tune investment

Per [model-lifecycle/fine-tuning-operations.md](../model-lifecycle/fine-tuning-operations.md):

- Curate the training set (often the few-shot examples are the seed; need 50–500+ for fine-tune).
- Run the fine-tune (provider-specific; varies in time and cost).
- Eval the fine-tuned model vs the few-shot baseline.
- Deploy the fine-tuned model with appropriate version pinning.

The investment is non-trivial; the payback is per-call savings + often a quality gain.

### 6.4 The hybrid: fine-tune + minimal few-shot

A fine-tuned model can still use a small number of examples (1–2) for edge-case calibration or as a sanity-check anchor. The hybrid captures the cost savings of fine-tune and the predictability of examples.

### 6.5 When fine-tune is wrong

- The use case isn't stable; the task keeps changing.
- The volume isn't high enough to justify the engineering investment.
- The team can't curate a quality training set.
- The provider's fine-tune offering has limitations the team can't work around.

In these cases, stay on few-shot. Re-evaluate quarterly.

### 6.6 The model-upgrade interaction

When the provider releases a new base model, the team's fine-tunes are based on the old. Decision:

- Re-fine-tune on the new base (cost: another fine-tune investment).
- Stay on the old base (cost: not getting the new model's improvements).

The decision is per-feature. Frequent base-model upgrades make fine-tuning more expensive; settle on a base-model release cadence the team can support.

---

## 7. Example set management at scale

When a team has 20+ prompts each with their own example sets, the management discipline matters.

### 7.1 The example registry

A central registry of examples used across prompts:

```yaml
examples:
  - id: clinical-summary-001
    prompt_versions: ["clinical-v3", "clinical-v4"]
    input: |
      ...
    output: |
      ...
    quality_score: 4.8  # judge score on this example
    last_audited: "2026-04-15"
    owner: clinical-team
    notes: "Anchors the SOAP note format"
```

The registry supports:

- Reuse across prompts.
- Quality tracking per example.
- Audit cadence.
- Ownership.

### 7.2 Example reuse vs duplication

Identical examples used in multiple prompts: reuse via the registry. Modifications to the example automatically apply to all prompts that use it.

Slight variants: each version is its own entry. Versions are linked via a parent/child relationship for traceability.

### 7.3 Quality-flagged examples

Examples with known issues (an output that's been corrected; an example that confuses the model on certain cases) are flagged. The flag prevents accidental reuse. The example's history is preserved for audit.

### 7.4 Ownership and review

Each example has an owner. Changes to the example go through the owner. Cross-team prompts (using the same examples) coordinate via the registry's ownership records.

### 7.5 The audit

Quarterly:

- Per-example: still relevant? Still high quality? Still useful?
- Per-prompt: is the example set's curve justified, or are some examples now redundant?
- Cross-prompt: are similar examples duplicated where they could be consolidated?

The audit removes ~5-10% of examples in a mature registry. Doesn't sound like much; over years, the discipline keeps the registry coherent.

### 7.6 Integration with prompt versioning

Per [prompt-versioning.md](./prompt-versioning.md), prompts are versioned artefacts. Example sets are part of the prompt's content; example changes are prompt changes; example changes go through the eval gate.

A prompt that "uses example X v3" is pinning a specific version of the example. Example updates that don't update the prompt's pin are not picked up. The team controls which examples each prompt version uses.

### 7.7 The "example becomes obsolete" lifecycle

When an example is no longer needed (the prompt evolved past it; the task changed; the example was redundant):

1. Mark the example deprecated in the registry.
2. Remove from all current prompts.
3. After a deprecation period, archive (preserve for history; not surfaced in active queries).

---

## 8. Eval and observability for few-shot

The eval surface for few-shot decisions.

### 8.1 Per-prompt eval

Standard prompt eval (per [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md)) covers prompt quality. Few-shot-specific:

- Curve analysis (per section 5.3).
- Example contribution measurement (which example contributes most / least).
- Negative-example testing (if used, do they help?).

### 8.2 Example-attribution eval

For retrieved examples, the eval traces which examples were used per case and correlates with outcome quality:

- "Cases where example X was retrieved scored Y on average."
- "Cases where the retrieval missed [pattern] failed at rate Z."

Used to identify example pool gaps and over-represented examples.

### 8.3 A/B testing example changes

Per [prompt-ab-testing.md](./prompt-ab-testing.md), example changes can be A/B tested:

- Variant A: current example set (5 examples).
- Variant B: pruned example set (3 examples).
- Outcome eval comparison; cost comparison.
- Promote variant B if quality holds and cost drops.

A/B testing example changes is faster than full prompt rewrites; supports incremental optimisation.

### 8.4 Observability of example impact

In the trace (per [agent-observability.md](../agent-engineering/agent-observability.md) when in agent context, or [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) more generally):

- Per-call span attribute: `prompt_version` (which captures the example set).
- For retrieved examples: `retrieved_examples` (list of example IDs).
- Token count attributes show example budget per call.

Aggregations support questions like "what fraction of call cost is example tokens?" — the data that drives the pruning audits.

### 8.5 Continuous eval

Production samples evaluated continuously surface drift. A change in input distribution (production traffic now includes more of a previously-rare case) may indicate the example set is no longer well-matched; the audit triggers earlier.

---

## 9. Worked Meridian example

Meridian's few-shot practice across the care-coordinator and patient-summary features.

### 9.1 Care-coordinator: minimal few-shot

The care-coordinator's main system prompt has 3 examples:

- Example 1: a clinical question with retrieval + answer + citation. Anchors the citation format.
- Example 2: a scheduling question with proposal + escalation. Anchors the escalation pattern.
- Example 3: a lookup question with simple direct answer. Anchors the "don't over-explain simple things" style.

Total example tokens: ~600. At ~$0.003 per call worth of example tokens at sonnet rates × 100k calls/day = ~$300/day in example cost. Worth it: eval shows 4% quality gain from these three examples vs zero examples.

Specific decisions:

- Curated, not retrieved. The care-coordinator's input shape is narrow enough (clinical questions, scheduling, lookups) that curation suffices.
- Three examples specifically — the curve analysis showed marginal gain drops to < 0.5% after example 3 on this prompt.
- The examples are in the example registry; owned by the clinical-content team; audited quarterly.

### 9.2 Patient-summary: zero few-shot

The patient-summary workflow's LLM steps (per [agent-vs-workflow-decision.md](../agent-engineering/agent-vs-workflow-decision.md) section 9.2) have well-defined inputs and outputs; the model handles them with instructions alone. The team experimented with adding examples; eval showed no improvement; the examples were not added.

The decision is recorded in the prompt's history ("examples tried, no benefit measured, omitted").

### 9.3 Analytics-warehouse copilot: dynamic few-shot

The SQL generation step in the analytics copilot uses retrieved examples:

- Example pool: 200 (question, SQL) pairs covering the warehouse's main analytical patterns.
- Retrieval: top-5 by embedding similarity of the user's question against the example question.
- Mix curated + retrieved: 2 curated examples (format anchoring) + 5 retrieved.

The variance in analyst questions (many different analytical shapes) justifies the retrieval infrastructure. Curated-only baseline scored 76% on eval; retrieved-hybrid scores 86%.

Example pool maintenance:

- New examples added when production reveals a pattern the retrieval misses.
- Quarterly pool audit; ~5% of pool refreshed per quarter.
- Per-example quality score; low-quality examples removed.

### 9.4 The example registry

~80 examples across all features. Owned variously by clinical-content, analytics, and platform teams. Audited quarterly.

Recent audit findings:

- 4 examples flagged for removal (no longer relevant; superseded by improved prompts).
- 2 examples flagged for revision (a clinical-content style update changed the canonical format).
- 1 example moved from "care-coordinator only" to "shared" (analytics copilot started using it for cross-functional questions).

### 9.5 The fine-tune decisions

Two of Meridian's features were examined for fine-tuning:

- **Patient-summary's clinical-history step.** ~25 examples in the few-shot prompt; high volume (~80k calls/day); stable use case. Fine-tuning analysis: projected $14k/year savings at the price of $3k initial + quarterly re-fine-tunes ($1k each = $4k/year). Net savings $10k/year. Decision: fine-tune. Implemented Q2-25; quality up 2%; cost down 60% per call; payback in ~5 months.
- **Analytics-copilot's SQL generation.** Considered but deferred — the retrieved-example pattern works well and would constrain the fine-tune's effectiveness; the team would need to choose one or the other. Currently sticking with retrieved few-shot.

### 9.6 What worked

- **Curve measurement before committing to examples.** Several proposed example additions were rejected when the curve showed flat marginal gain.
- **Per-feature decisions, not org-wide policy.** Care-coordinator uses 3 examples; analytics uses retrieval; patient-summary uses zero. Each is right for its task.
- **Example registry.** Reuse across features; consistent ownership; audit cadence.
- **Fine-tune when justified.** The patient-summary fine-tune was the largest single cost reduction in the past year.

### 9.7 What didn't work initially

- **Reflexive example addition.** Early prompts grew to 15+ examples; audit revealed half were past the curve's knee. Pruned.
- **Example duplication across prompts.** Same example was copy-pasted into multiple prompts; updates diverged; quality varied. Registry consolidation fixed this.
- **No example ownership.** Early on, examples were "everyone's responsibility" → "no one's responsibility." Owner assignment forced accountability.

---

## 10. Anti-patterns

### 10.1 "Add examples until it works; never revisit"

The team adds examples to fix observed failures; never measures whether each example justifies its cost; never prunes.

**Corrective.** Curve measurement per section 5.3; quarterly audit per section 5.4.

### 10.2 "Examples not in eval"

Example additions ship without eval support. Whether they actually help is unknown.

**Corrective.** Every example addition justified by eval delta. PR-gated.

### 10.3 "Examples drift from production format"

Examples were written when the input format was different; production has evolved; examples no longer match.

**Corrective.** Examples reflect current production format. Audit catches drift.

### 10.4 "Retrieved examples without diversity"

Top-N-similar always returns examples from the same cluster; the model's behaviour narrows.

**Corrective.** Diversity in retrieval per section 3.5; MMR or similar.

### 10.5 "Duplicated examples across prompts"

Same example is copy-pasted into 5 prompts; updates diverge; behaviour inconsistent.

**Corrective.** Example registry per section 7.1; reuse via registry.

### 10.6 "Examples without owners"

Examples accumulate; no one is responsible; nothing is audited or removed.

**Corrective.** Per-example owner per section 7.4.

### 10.7 "Fine-tune for everything"

The team fine-tunes every prompt; many tasks are small-volume and the fine-tune investment doesn't pay back.

**Corrective.** Per-section-6 criteria; fine-tune only when justified.

### 10.8 "Negative examples confuse the model"

Negative examples were added but the model interpreted them as positive examples; quality dropped.

**Corrective.** Negative examples used sparingly; tested against eval; removed if they don't help.

---

## 11. Findings (sprint-assignable)

### FEW-SHOT-001 — Severity: Critical
**Finding.** High-volume feature's prompt has > 15 examples without curve measurement; example cost significant fraction of feature cost.
**Recommendation.** Curve analysis per section 5.3; prune to the knee; measure cost savings.
**Owner.** ai-platform-eng + feature-team, sprint N+1.

### FEW-SHOT-002 — Severity: High
**Finding.** Examples have been added historically without eval support; impact unknown.
**Recommendation.** Eval each example per section 5.3; remove examples with negligible contribution.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### FEW-SHOT-003 — Severity: High
**Finding.** Same example duplicated across multiple prompts; updates inconsistent.
**Recommendation.** Example registry per section 7.1; consolidate duplicates.
**Owner.** ai-platform-eng, sprint N+2.

### FEW-SHOT-004 — Severity: High
**Finding.** Retrieved examples lack diversity in retrieval; example pool over-retrieved from narrow cluster.
**Recommendation.** Diversity per section 3.5; MMR or similar; eval impact.
**Owner.** ai-platform-eng, sprint N+2.

### FEW-SHOT-005 — Severity: Medium
**Finding.** Example set was never re-audited; some examples may be obsolete.
**Recommendation.** Quarterly audit per section 5.4 / 7.5.
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### FEW-SHOT-006 — Severity: Medium
**Finding.** No example ownership; examples accumulate without responsibility.
**Recommendation.** Per-example owner per section 7.4.
**Owner.** ai-platform-eng, sprint N+3.

### FEW-SHOT-007 — Severity: Medium
**Finding.** Stable high-volume feature with 25+ examples; fine-tuning not examined.
**Recommendation.** Fine-tune analysis per section 6; implement if justified.
**Owner.** ai-platform-eng + ml-eng, sprint N+3.

### FEW-SHOT-008 — Severity: Medium
**Finding.** Example format inconsistent with production input format; potential drift effect.
**Recommendation.** Format alignment per section 4.4; eval after alignment.
**Owner.** ai-platform-eng, sprint N+3.

### FEW-SHOT-009 — Severity: Medium
**Finding.** Example token cost not tracked; cost optimisation invisible.
**Recommendation.** Per-call example token cost in attribution per [cost-attribution.md](../cost-and-finops/cost-attribution.md); dashboard.
**Owner.** ai-platform-eng, sprint N+3.

### FEW-SHOT-010 — Severity: Medium
**Finding.** Retrieved example pool not curated; low-quality historical examples used.
**Recommendation.** Pool curation per section 3.5; periodic pool audit.
**Owner.** ai-platform-eng + feature-team, sprint N+4.

### FEW-SHOT-011 — Severity: Medium
**Finding.** Examples not in version control; changes untracked.
**Recommendation.** Registry under version control per section 7.1; PR review for changes.
**Owner.** ai-platform-eng, sprint N+4.

### FEW-SHOT-012 — Severity: Medium
**Finding.** New examples not A/B tested before broad deployment.
**Recommendation.** A/B test per [prompt-ab-testing.md](./prompt-ab-testing.md); promote on signal.
**Owner.** ai-platform-eng, sprint N+4.

### FEW-SHOT-013 — Severity: Medium
**Finding.** Examples for smaller-model tier not differentiated from frontier-model tier.
**Recommendation.** Per-tier example sets per section 2.6; smaller models often need more.
**Owner.** ai-platform-eng, sprint N+4.

### FEW-SHOT-014 — Severity: Low
**Finding.** Example labels missing or inconsistent (some labelled "Example 1", some not labelled).
**Recommendation.** Consistent labelling per section 4.6.
**Owner.** ai-platform-eng, sprint N+4.

### FEW-SHOT-015 — Severity: Low
**Finding.** Negative examples included but not tested; may hurt rather than help.
**Recommendation.** Test per section 4.3; remove if not helping.
**Owner.** ai-platform-eng, sprint N+5.

### FEW-SHOT-016 — Severity: Low
**Finding.** Example ordering not deliberate; strong examples buried mid-list.
**Recommendation.** Order per section 4.1; eval the ordering change.
**Owner.** ai-platform-eng, sprint N+5.

### FEW-SHOT-017 — Severity: Low
**Finding.** Example registry not searchable; reuse across teams difficult.
**Recommendation.** Searchable registry per section 7.1; tagged by domain, task.
**Owner.** ai-platform-eng, sprint N+5.

### FEW-SHOT-018 — Severity: Low
**Finding.** Fine-tune re-evaluation not scheduled; previous fine-tune may be obsolete after base-model upgrade.
**Recommendation.** Re-evaluation cadence per section 6.6; quarterly.
**Owner.** ai-platform-eng + ml-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team using few-shot:

- [ ] **Sprint 0 — audit.** What features use few-shot? How many examples each? Cost contribution per feature.
- [ ] **Sprint 1 — curve analysis.** Per feature: measure marginal gain of each example. Identify candidates for removal.
- [ ] **Sprint 1 — pruning.** Remove past-the-knee examples; verify quality stable; capture cost savings.
- [ ] **Sprint 2 — example registry.** Build (if absent); migrate duplicates to consolidated entries.
- [ ] **Sprint 2 — ownership.** Each example has an owner.
- [ ] **Sprint 3 — eval discipline.** Every example addition justified by eval delta.
- [ ] **Sprint 3 — fine-tune candidates.** Identify features meeting fine-tune criteria; analyse ROI.
- [ ] **Sprint 4 — retrieved-example evaluation.** For high-variance input features; consider hybrid pattern.
- [ ] **Ongoing — quarterly audit.** Per-example relevance, quality, contribution; registry hygiene.

For a team starting few-shot:

- [ ] **Sprint 0 — zero-shot baseline.** Build the prompt without examples; eval; identify failure modes.
- [ ] **Sprint 1 — add examples one at a time.** Eval after each; stop at the curve's knee.
- [ ] **Sprint 1 — register the examples.** Owner, registry entry, version.
- [ ] **Sprint 2 — observability.** Token cost tracked; per-feature dashboard.
- [ ] **Sprint 2 — audit cadence.** Quarterly review scheduled.

A team that completes the sequence has few-shot examples that earn their tokens, are maintained, and are pruned when no longer justified. A team that doesn't has prompts that bloat over time and a cost line that grows without quality improvement.

---

## 13. References

- [prompts-as-code-discipline.md](./prompts-as-code-discipline.md) — examples are part of the prompt; same versioning + ownership.
- [prompt-versioning.md](./prompt-versioning.md) — example changes are prompt changes; gate appropriately.
- [structured-output-engineering.md](./structured-output-engineering.md) — schema constraints + few-shot for format anchoring.
- [prompt-libraries-and-components.md](./prompt-libraries-and-components.md) — example sets as reusable components.
- [prompt-ab-testing.md](./prompt-ab-testing.md) — A/B testing example changes.
- [prompt-as-api-contract.md](./prompt-as-api-contract.md) — example changes that affect output format are contract changes.
- [prompt-anti-patterns.md](./prompt-anti-patterns.md) — including "kitchen-sink prompts" and "ungrounded few-shot."
- [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — eval practice that surfaces example-change impact.
- [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — golden sets used to measure example contribution.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — per-call token cost includes example tokens; visible in attribution.
- [cost-and-finops/tier-routing-for-cost.md](../cost-and-finops/tier-routing-for-cost.md) — per-tier few-shot decisions.
- [model-lifecycle/fine-tuning-operations.md](../model-lifecycle/fine-tuning-operations.md) — fine-tune operational depth.
- [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — per-call token attribution.
- Sibling repo: [ai-architecture-reference-architecture/context-and-prompt-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture) — architectural framing of context (planned).
- Anthropic / OpenAI prompt-engineering documentation — provider-specific few-shot guidance.
- "Language Models are Few-Shot Learners" (Brown et al., 2020) — foundational paper on in-context few-shot capability.
