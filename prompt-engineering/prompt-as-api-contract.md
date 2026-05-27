# Prompt as API Contract

> **Audience.** Engineers and tech leads whose prompts have downstream consumers — parsers, evals, structured extractors, chained prompts, UI components rendering the output. Anyone whose "we just need to tweak the prompt a little" turned into a multi-team incident because consumers broke. **Scope.** The engineering discipline of treating a prompt's output as a versioned API contract — breaking change classification, deprecation lifecycle, consumer-driven contract testing, the multi-consumer coordination patterns. Not the prompt-versioning mechanics (see [prompt-versioning.md](./prompt-versioning.md)). Not the structured output engineering depth (see [structured-output-engineering.md](./structured-output-engineering.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

A prompt is an interface. The model is the implementation. The output is the result of the interface call. Everything downstream of the prompt — parsers that extract structured fields, evals that score the output, formatters that render it, subsequent prompts that consume it, UIs that display it, side-effecting code that acts on it — is a *consumer* of the prompt's output contract.

When the prompt changes, the contract can change. Sometimes the change is innocuous (a wording adjustment that doesn't affect downstream consumers). Sometimes it's a breaking change in disguise: the output's JSON structure adds a field consumers parse with `dict[key]` and now fail; the output's tone shifts and the eval's judge scores it differently; the output's length changes and the UI's rendering breaks at edge cases.

Most teams treat prompt changes as "just text edits" — small, low-risk, ship anytime. The framing is wrong for any prompt with non-trivial consumers. A prompt change is a *contract change* with the same operational characteristics as an API change: backward compatibility matters, deprecation is a lifecycle, consumer testing matters, coordination is needed.

This document is opinionated about four things:

1. **A prompt with downstream consumers is an API.** Treat it that way. Versioned, documented, contract-tested, deprecated through a lifecycle.
2. **Breaking changes are classified explicitly.** "Major" vs "minor" vs "patch" is not a stylistic choice; it's an operational fact that determines the consumer-coordination burden.
3. **Consumers drive the contract.** Consumer-driven contract tests verify the prompt produces what consumers depend on. The prompt's owner doesn't ship breaking changes without consumer migration.
4. **Deprecation is a lifecycle, not an event.** Old versions are deprecated, migrated, removed. Forced removal without migration creates incidents.

Structure: (2) what makes a prompt an API; (3) the output as contract — what consumers depend on; (4) breaking change classification; (5) the deprecation lifecycle; (6) consumer-driven contract tests; (7) multi-consumer coordination patterns; (8) versioning interaction with structured output; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. What makes a prompt an API

The contract perspective.

### 2.1 The input contract

The prompt has an input contract: what's expected to be passed in.

For a templated prompt:

- The template variables (`{user_input}`, `{patient_id}`, `{retrieved_context}`).
- The types or shapes of each variable.
- The constraints (length limits, required vs optional, format requirements).

Callers depend on the input contract. A prompt that suddenly requires a new variable breaks existing callers; this is a breaking change to the input contract.

### 2.2 The output contract

The prompt has an output contract: what the model is expected to produce.

For a structured-output prompt (per [structured-output-engineering.md](./structured-output-engineering.md)):

- The schema (fields, types, formats, required vs optional).
- The value constraints (enums, ranges).
- The semantic meaning of each field.

For a free-text prompt:

- The general format (paragraphs, bullet lists, sections).
- The style/tone characteristics consumers rely on.
- The structural conventions (citation format, heading style).
- The expected length range.

Consumers depend on the output contract. A change to the schema or to the rendering conventions can break consumers.

### 2.3 The behavioural contract

Beyond input and output shape, the prompt has a behavioural contract:

- What does it do (its purpose, intent, scope).
- What it doesn't do (refusal patterns, escalation behaviour).
- What inputs produce what kinds of outputs.

Consumers may depend on behavioural promises. A change in behaviour (e.g., the prompt now refuses what it used to handle) is a contract change even if input and output schemas are unchanged.

### 2.4 The "but it's just a prompt" pushback

A common pushback to the API-contract framing: "the prompt is just text, not code; we can change it freely."

The framing is wrong because the *consumers* don't care about the source. They care about what comes out. If a downstream parser expects a JSON field `patient_id` and the prompt's update removed that field from the model's output, the parser breaks. The fact that the change was "just text" doesn't help the parser.

The API-contract framing isn't an abstraction; it's a description of how the prompt actually behaves with respect to its consumers.

### 2.5 Where the contract perspective starts to matter

A prompt with one consumer (the immediate calling code) and no downstream complexity is barely an API; treat it lightly. A prompt with:

- Multiple consumers (parsers, evals, chained prompts, UI).
- Cross-team consumers.
- Structured output that downstream code depends on.
- Eval graders that depend on output shape.
- Production traffic that would surface regressions.

— warrants the API-contract treatment. Most production prompts of any complexity fit.

### 2.6 The cost of contract treatment

It is real. PR review involves consumer impact analysis. Eval gates include consumer tests. Deprecation lifecycles require coordination. The investment isn't free.

For prompts that don't have multiple consumers or cross-team dependencies, the lighter touch is appropriate. The discipline scales with the prompt's blast radius.

---

## 3. The output as contract — what consumers depend on

The output contract has multiple layers; each can be the source of a breaking change.

### 3.1 The schema layer

For structured output:

- Field presence (which fields are always present, which are optional).
- Field types.
- Field formats (dates, UUIDs, enums).
- Nested structures.
- Array lengths and constraints.

Consumers depend on the schema; a missing field or a type change breaks them.

### 3.2 The semantic layer

The schema can be the same but the *meaning* of fields can change. The field `cost_estimate` could have meant "best estimate" in v1 and "worst case" in v2; consumers that summed them produce wrong totals.

Semantic changes are particularly insidious because they're invisible in type-only validation; they require behavioural tests.

### 3.3 The format layer

For free-text or partially-structured output:

- Section headers (consumers may parse based on section names).
- Markdown vs plain text.
- Citation format.
- Bullet vs numbered lists.

UI consumers often parse these conventions; changes break the rendering.

### 3.4 The style layer

Tone, length, formality. Consumers (judges in evals, summarisers, chained prompts) may depend on the style.

Style changes are subjective and harder to detect; their effect may surface in eval scores rather than parser errors.

### 3.5 The behaviour layer

What the prompt actually does:

- What inputs produce what kinds of outputs.
- Refusal patterns.
- Escalation behaviour.
- Edge case handling.

Consumer tests cover behavioural expectations; "if input is X, output should contain Y" — these are contract tests.

### 3.6 The implicit contract

A specific risk: consumers come to depend on properties the prompt's author didn't intend as guarantees. The prompt happens to output certain things in a certain order; consumers parse based on order; the prompt's author never thought of order as load-bearing; the next change shuffles order; consumers break.

The lesson: the contract is what consumers depend on, not what the author intends. Implicit dependencies are still dependencies. Consumer testing surfaces them.

### 3.7 The mitigation: explicit contract documentation

For non-trivial prompts:

- A contract specification document.
- Lists the input variables, types, constraints.
- Lists the output schema (or structural conventions).
- Lists behavioural guarantees ("if X, output contains Y"; "if input lacks Z, output indicates").
- Lists explicit non-guarantees ("field ordering is not stable"; "exact phrasing varies").

The non-guarantees prevent consumers from accidentally depending on volatile properties.

---

## 4. Breaking change classification

Per [prompt-versioning.md](./prompt-versioning.md), semantic versioning. The classification:

### 4.1 Major (breaking) changes

Changes that consumers must adapt to:

- Removing a field from structured output.
- Renaming a field.
- Changing a field's type.
- Tightening a field's constraints (e.g., shorter max length than before).
- Changing the meaning of a field.
- Removing a behavioural guarantee.
- Removing or significantly altering refusal/escalation patterns consumers rely on.
- Removing a section from a multi-section free-text output.

These require consumer migration. Version bumps major (v1.x → v2.0).

### 4.2 Minor (non-breaking) changes

Changes that are additive or improve quality without breaking consumers:

- Adding a new field to structured output (optional; consumers ignoring it are unaffected).
- Adding a new section to multi-section free-text (consumers parsing known sections are unaffected).
- Improving content quality without changing shape (the citation is more accurate; the wording is better).
- Adding a new behaviour (a new edge case handled; a new refusal scenario added).

Minor bumps (v1.2 → v1.3). Consumers can adopt at their own pace; old code still works with new output.

### 4.3 Patch changes

Backward-compatible fixes:

- Typo corrections.
- Small wording improvements that don't affect semantics.
- Better handling of a previously-broken edge case.
- Performance/cost optimisations that don't change observable behaviour.

Patch bumps (v1.2.3 → v1.2.4). Consumers should be unaffected.

### 4.4 The classification difficulty

Some changes are ambiguous:

- "Tightening" a behavioural guarantee — is this major (because some inputs that worked now don't) or minor (because the prompt is now more correct)?
- Style changes — minor (no schema change) or major (consumers that depend on style break)?
- Refusal pattern adjustments — major if previously-handled cases now refuse; minor if previously-uncovered cases now refuse correctly.

The discipline: when in doubt, classify higher (more breaking). Forcing the more-conservative classification protects consumers and triggers the proper coordination.

### 4.5 The breaking-change-without-version-bump anti-pattern

Some teams make breaking changes without bumping the major version. Consumers don't notice the version pin doesn't protect them; the change ships; consumers break.

This is a discipline failure, not a technical limitation. The classification has to be honest.

### 4.6 The "non-breaking on paper, breaking in practice" failure

A change is technically additive (adds a new field) but the new field changes the output's character (the model now spends tokens on the new field, producing shorter content for existing fields; consumers' UI breaks because expected content length differs).

The contract perspective: the change broke a property consumers depended on (even if implicitly). Major bump warranted.

### 4.7 The "I tested the consumers" defence

A prompt author who tests all known consumers and verifies they still work may believe they avoided a breaking change. They may be right. But unknown consumers (a team that integrated the prompt's output last quarter; an internal tool nobody remembers) may break.

The contract is the surface of the prompt's output, not the surface of known consumers. Treat changes per the classification, not per the test results.

---

## 5. The deprecation lifecycle

Old versions don't disappear; they retire through a defined process.

### 5.1 The lifecycle stages

1. **Active.** The current version. Most consumers use it.
2. **Stable.** Older version still in use; supported; bugs fixed; new features not added.
3. **Deprecated.** Marked for removal; consumers asked to migrate; supported for the deprecation period.
4. **Removed.** No longer available; consumers must have migrated.

The lifecycle parallels API deprecation. Same discipline.

### 5.2 The deprecation period

How long the deprecation period is depends on:

- The number and importance of consumers.
- The complexity of consumer migration.
- The risk of leaving the old version in place.

Typical periods:

- **Internal-only prompts:** 1–3 months.
- **Cross-team prompts:** 3–6 months.
- **Stable, widely-used prompts:** 6–12 months.

The deprecation period is decided at the time of deprecation, communicated, and adhered to.

### 5.3 The deprecation announcement

When a version is deprecated:

- Documented in the prompt's changelog.
- Announced to consumers (team chat, email, dashboard).
- Migration path provided (what to change to use the new version).
- Deprecation date communicated.
- Removal date communicated.

The communication clarity is what enables consumer migration.

### 5.4 The migration support

The prompt's author / owning team supports consumer migration:

- Office hours / availability for questions.
- Specific migration examples.
- Help with consumer-side eval if needed.

The support is part of the discipline. Forcing migration without support strains relationships.

### 5.5 The "consumer that didn't migrate" failure mode

The removal date arrives; some consumers haven't migrated. Options:

- **Hard removal.** The old version is gone; non-migrated consumers break. Sharp; punishing of slow consumers; sometimes necessary.
- **Extension.** Grant additional time; communicate clearly that this is the last extension.
- **Soft removal.** The old version is removed from the active registry but kept in archive; the consumer's pin still works but is flagged as deprecated.

Pick the option that fits the risk and the relationships.

### 5.6 The forced deprecation

For safety, compliance, or security reasons, a version may need to be deprecated faster than consumers can migrate:

- Hot-patch the old version with the safety fix (if possible).
- Force consumers to a fixed deadline.
- Provide intense migration support.
- Document the forced deprecation in the post-incident review.

Forced deprecations are unusual; they strain consumer relationships; reserve for genuine necessity.

### 5.7 The version-pin behaviour at removal

When a removed version is referenced (the consumer's pin still points at it):

- The composer (per [prompt-libraries-and-components.md](./prompt-libraries-and-components.md)) fails loudly.
- The CI build fails.
- The runtime — if reached — produces a structured error.

Loud failure is preferable to silent fallback (which would behave differently than expected).

---

## 6. Consumer-driven contract tests

The technique adapted from API contract testing.

### 6.1 What consumer-driven contract tests are

Each consumer of the prompt declares what they expect from the prompt's output:

- "I parse field X; it must be present."
- "I expect output to be valid JSON matching this schema."
- "I expect this section to appear."
- "I expect the response length to be < N tokens."

The consumer's tests run against the prompt's current output; if the prompt's output doesn't satisfy a consumer's contract, the prompt's CI fails.

### 6.2 The benefit

The prompt's author finds out about consumer breakage at PR time, not at production deploy time. The PR is updated (either to preserve the contract or to bump the major version) before merging.

The benefit compounds: consumer-driven tests catch implicit contract dependencies that explicit specifications miss. The consumer's tests are the ground truth of what the consumer relies on.

### 6.3 The implementation

For each consumer of a prompt:

```python
# In consumer's test suite, but consumed by the prompt's CI:
class CareCoordinatorContract(PromptContract):
    prompt_id = "care-coordinator-system"
    
    def test_output_is_valid_json_with_required_fields(self, sample_outputs):
        for output in sample_outputs:
            data = json.loads(output)
            assert "patient_id" in data
            assert "response" in data
    
    def test_response_length_under_limit(self, sample_outputs):
        for output in sample_outputs:
            data = json.loads(output)
            assert len(data["response"]) < 2000
```

The prompt's CI runs all consumer contracts against a sample of outputs from the proposed prompt change. Failures block the PR.

### 6.4 The sample-output corpus

The contract tests need outputs to test against. Sources:

- The pre-prod eval set (the prompt's standard eval input cases).
- Production samples (per [agent-observability.md](../agent-engineering/agent-observability.md)).
- Synthetic edge cases.

A few hundred sample outputs is typically enough to surface contract violations.

### 6.5 The "consumer test is too strict" failure

A consumer test that asserts "output always starts with 'Hello'" is too strict; many improvements to the prompt would fail. The consumer test should test what the consumer needs, not over-specify.

Reviews of consumer tests reduce over-specification. The consumer owns the test; the prompt's owner can request revisions if the test is too strict.

### 6.6 The "consumer test is too lax" failure

A consumer test that asserts only "output is non-empty" doesn't protect against meaningful regressions. Better tests assert the specific things the consumer depends on.

The discipline: consumer tests test the consumer's actual dependency, with appropriate specificity.

### 6.7 The contract-testing infrastructure

For mature teams:

- Each prompt's CI knows its consumers.
- Each consumer maintains contract tests in their repo (or in a shared contracts repo).
- The CI fetches and runs all consumer tests.
- Test failures explain which consumer breaks and what to do.

The infrastructure investment is moderate; the value compounds as consumer count grows.

---

## 7. Multi-consumer coordination patterns

When a prompt has many consumers across teams, coordination matters.

### 7.1 The consumer registry

A registry of which teams/services consume each prompt:

```yaml
prompt: care-coordinator-system
consumers:
  - team: care-coordinator-frontend
    contact: alice@meridian.com
    contracts: ["valid-json", "response-length", "patient-id-present"]
  - team: care-coordinator-eval
    contact: bob@meridian.com
    contracts: ["eval-set-passes"]
  - team: care-coordinator-analytics
    contact: carol@meridian.com
    contracts: ["analytics-fields-present"]
```

The registry supports communication and impact analysis.

### 7.2 Impact analysis per change

Before a prompt change ships:

- Identify which consumers are affected (from the registry).
- Run each affected consumer's contract tests.
- For breaking changes, notify consumers in advance.

The discipline ensures no consumer is surprised.

### 7.3 The change-coordination process

For major changes:

1. Prompt author drafts the new version.
2. Identifies consumers from the registry.
3. Notifies consumers; collects feedback.
4. Consumers review; provide input; update their contracts if needed.
5. Prompt author finalises; ships with new major version; old version deprecated.
6. Consumers migrate at their pace within the deprecation window.

### 7.4 The "no consumer is responsive" failure

Some consumers are unresponsive — busy teams, ambiguous ownership, holiday season. Prompt change blocked indefinitely waiting.

Mitigation:

- Set response deadlines in the change-coordination process (e.g., 5 business days).
- Escalate unresponsive consumers to their team lead.
- Proceed with change after deadline if no response; consumers responsible for catching up.

### 7.5 The cross-team prompt's special burden

Prompts used across teams have heavier coordination burden than single-team prompts. The team that owns the prompt bears the coordination cost. Some teams resist owning cross-team prompts because of this.

Mitigation: dedicated cross-team prompt owners (often platform team). The cost is real; budget for it.

### 7.6 The chained-prompt consumer

Some "consumers" are themselves prompts: prompt B takes prompt A's output as input. Prompt A's contract change can break prompt B.

Treat prompt B as a consumer of prompt A. Same coordination process. The chain produces transitive dependencies; tracking these matters.

For long chains (3+ prompts in a pipeline), consider whether the chain is the right design vs a workflow with explicit steps (per [agent-vs-workflow-decision.md](../agent-engineering/agent-vs-workflow-decision.md)).

---

## 8. Versioning interaction with structured output

When the prompt produces structured output (per [structured-output-engineering.md](./structured-output-engineering.md)), the schema is part of the contract.

### 8.1 The schema as explicit contract

For structured-output prompts, the schema is the most explicit part of the contract. It's machine-readable, testable, and unambiguous.

The schema's version is the prompt's version (or evolves with it). Adding a field is a minor bump. Removing or renaming is a major bump. Changing a type is a major bump.

### 8.2 Schema evolution patterns

Per general API design:

- **Additive changes are safe.** New optional fields, new enum values (in many cases).
- **Renames need both old and new for a deprecation window.** Field is present under both names; consumers migrate to new name; old name removed after window.
- **Removals need consumer migration first.** Consumers stop using the field; then field is removed.

The structured-output schema is amenable to API-evolution best practices.

### 8.3 Strict schema enforcement

Per [structured-output-engineering.md](./structured-output-engineering.md), strict schemas (rejected outputs that don't match) are the enforcement mechanism. The schema's strictness is itself a contract: consumers can rely on the output being schema-conformant.

Relaxing the schema (allowing previously-rejected outputs) is potentially a breaking change for consumers that depended on the rejection (rare but possible).

### 8.4 Schema versioning across consumers

When multiple consumers of the same prompt's structured output exist:

- The schema is one (the prompt produces one shape).
- Consumers can use different subsets (consumer A reads fields x, y; consumer B reads y, z).
- A field used by only one consumer can still be removed if that one consumer migrates.

The coordination is per-field: who uses what? The registry helps.

### 8.5 Free-text within structured output

Sometimes structured output has free-text fields (e.g., `response_text: string`). The free text is still subject to behavioural contract — tone, length, content expectations.

Free-text fields are softer contracts than schema fields; consumers may parse the free text (header detection, markdown parsing) and become brittle. Discourage this; if consumers need structure, add structured fields.

---

## 9. Worked Meridian example

Meridian's prompt-as-API-contract practice.

### 9.1 The contract registry

`meridian-prompt-contracts` repository holds:

- Per-prompt contract documentation.
- Consumer registry per prompt.
- Consumer contract tests.

Maintained by the prompt-library team; updated alongside prompts.

### 9.2 An example contract document

`care-coordinator-system.md`:

```
# Care-Coordinator System Prompt Contract

## Input
Variables: {user_input}, {patient_context}, {tenant_id}
Constraints: {patient_context} max 4000 chars; {user_input} max 2000 chars.

## Output
Structured output schema (Anthropic tool use): see schema.json
Fields:
- response: string (the user-facing response; max 1500 chars)
- escalate: boolean
- escalation_reason: string (required if escalate=true; max 500 chars)
- tools_called: list of strings (the tools the agent used)
- citations: list of {doc_id: string, section: string}

## Behavioural guarantees
- If the user asks for medical advice, escalate=true.
- If the user asks about another patient (mismatched patient_id), escalate=true.
- If retrieval found no relevant context, response contains a "no relevant information" indicator.
- response field uses clinical tone.
- citations include all document references used in response.

## Non-guarantees
- Exact phrasing varies.
- citation order is not stable.
- response sentence count is not stable.
```

### 9.3 The consumer registry

```yaml
prompt: care-coordinator-system
version: v23
consumers:
  - team: care-coordinator-frontend
    contact: frontend-team@meridian.com
    contracts: ["response-field-present", "escalation-fields-consistent", "citation-format"]
  - team: care-coordinator-eval
    contact: ai-platform@meridian.com
    contracts: ["eval-golden-set-passes", "no-pii-in-response"]
  - team: care-coordinator-analytics
    contact: analytics@meridian.com
    contracts: ["tools-called-field-present", "escalation-reason-when-escalated"]
  - team: care-coordinator-audit
    contact: compliance@meridian.com
    contracts: ["citations-present-for-clinical-content"]
```

Four consumers; each with specific contracts. Coordination matters.

### 9.4 A breaking change (Q4-25)

The team wanted to restructure the output:

- Remove `tools_called` (it duplicated information in the trace; consumers could get it from traces).
- Add `confidence_score: float` (new field).
- Change `citations` from `list of {doc_id, section}` to `list of {doc_id, section, relevance: float}` (added field within nested structure).

Classification: major bump (v23 → v24) because `tools_called` removal breaks consumers.

Process:

1. Author drafted v24 spec.
2. Identified 4 consumers from registry.
3. Care-coordinator-analytics relied on `tools_called`; coordinated migration to read from traces instead.
4. Other 3 consumers were unaffected (didn't use `tools_called`; new fields were additive for them).
5. v24 released; v23 deprecated with 90-day window.
6. Analytics team migrated; verified contract tests pass; updated their pin to v24 within 30 days.
7. After 90 days, v23 removed from active registry.

The coordination took ~6 weeks calendar time including the migration window. Without the contract discipline, the change would have broken analytics in production.

### 9.5 A consumer-test catching a near-miss (Q1-26)

A proposed minor update to the care-coordinator's prompt was meant to be backward-compatible — improved wording, no schema change.

The consumer test for care-coordinator-frontend asserted "response field max 1500 chars." The proposed change shifted the prompt's verbosity; the output now sometimes exceeded 1500 chars; the consumer test caught it in CI.

The author's options:

- Tighten the prompt to ensure 1500-char limit holds.
- Bump major version and coordinate with frontend to support longer responses.

The author chose option 1; tightened the prompt; CI passed. The frontend never broke.

Without the consumer test, the change would have shipped; consumer regressions would have appeared in production rendering.

### 9.6 The deprecation lifecycle worked

Three deprecations over 14 months. Each followed the lifecycle:

- Announcement.
- Migration path documented.
- Office hours for consumer questions.
- Consumers migrated within the window.
- Old versions removed.

No consumer-side production incident from a removed version in 14 months.

### 9.7 The infrastructure

- Contract tests in Pytest; CI runs them on every prompt PR.
- Consumer registry as YAML; PR-reviewed when changes.
- Contract docs alongside prompts.
- Prompt store enforces version pins; removed versions error loudly.

### 9.8 What didn't work initially

- **Implicit contracts.** Early prompts didn't have explicit contract docs; consumers' implicit dependencies caused breakage. Adopted explicit docs after.
- **No consumer registry.** Coordination was ad-hoc; contributors forgot consumers. Registry centralised tracking.
- **Forced deprecations.** Two early "we need this gone" cases caused consumer pain. Tightened the deprecation lifecycle to formal stages.

### 9.9 The cost

The contract discipline costs ~10% of prompt-engineering time (specification, coordination, contract tests). Recovered many times over in reduced incidents and faster cross-team work.

---

## 10. Anti-patterns

### 10.1 "Prompt is just text; change freely"

Prompts are edited without consideration of consumers. Breaking changes ship; consumers break.

**Corrective.** API-contract perspective per section 2; classify changes per section 4.

### 10.2 "No version bump on breaking change"

Breaking changes ship without major version bump; consumer pins don't protect them.

**Corrective.** Honest classification per section 4.5; major bumps when needed.

### 10.3 "No consumer registry"

Coordination is ad-hoc; some consumers are forgotten; they break.

**Corrective.** Consumer registry per section 7.1.

### 10.4 "No contract documentation"

Consumers depend on implicit properties; no written contract; changes break unforeseen things.

**Corrective.** Contract docs per section 3.7.

### 10.5 "Hard removal without migration"

Old version removed before consumers migrated; consumers break in production.

**Corrective.** Deprecation lifecycle per section 5.

### 10.6 "Consumer tests too strict"

Tests assert things that aren't real contracts; reasonable improvements break them.

**Corrective.** Right specificity per section 6.5; consumer-side review.

### 10.7 "Cross-team prompts without dedicated owner"

Cross-team prompts have no clear owner; coordination falls through gaps.

**Corrective.** Dedicated owner per section 7.5.

### 10.8 "Chained-prompt dependency not tracked"

Prompt B consumes prompt A's output; prompt A's change breaks B; nobody noticed in advance.

**Corrective.** Chained prompts as consumers per section 7.6; in the registry.

---

## 11. Findings (sprint-assignable)

### PROMPT-API-001 — Severity: Critical
**Finding.** Breaking prompt changes ship without major version bump; consumers break in production.
**Recommendation.** Classification discipline per section 4; honest major bumps.
**Owner.** ai-platform-eng + tech leads, sprint N+1.

### PROMPT-API-002 — Severity: Critical
**Finding.** No consumer registry; consumers forgotten; coordination fails.
**Recommendation.** Consumer registry per section 7.1; PR-reviewed.
**Owner.** ai-platform-eng, sprint N+1.

### PROMPT-API-003 — Severity: Critical
**Finding.** Old prompt versions removed without deprecation; consumers break.
**Recommendation.** Deprecation lifecycle per section 5; standard windows.
**Owner.** ai-platform-eng, sprint N+1.

### PROMPT-API-004 — Severity: High
**Finding.** Contract documentation absent; consumers' implicit dependencies untracked.
**Recommendation.** Per-prompt contract docs per section 3.7.
**Owner.** ai-platform-eng + feature teams, sprint N+2.

### PROMPT-API-005 — Severity: High
**Finding.** No consumer-driven contract tests; consumer breakage detected only in production.
**Recommendation.** Contract tests per section 6; CI-integrated.
**Owner.** ai-platform-eng + consumer teams, sprint N+2.

### PROMPT-API-006 — Severity: High
**Finding.** Chained-prompt dependencies not tracked; breaking changes cascade unexpectedly.
**Recommendation.** Chained prompts in registry per section 7.6; impact analysis covers chain.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-API-007 — Severity: High
**Finding.** Forced deprecations strain consumer relationships; non-emergency cases use the forced path.
**Recommendation.** Use forced deprecation only for safety/compliance per section 5.6.
**Owner.** ai-platform-eng + leadership, sprint N+2.

### PROMPT-API-008 — Severity: Medium
**Finding.** Cross-team prompts without dedicated owners; coordination falls through.
**Recommendation.** Owners per section 7.5; platform team often owns cross-team prompts.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-API-009 — Severity: Medium
**Finding.** Structured-output schema changes don't follow API evolution patterns; breaking renames ship without coexistence period.
**Recommendation.** Schema-evolution patterns per section 8.2.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-API-010 — Severity: Medium
**Finding.** Removed-version references fail silently (default behaviour); consumers get unexpected output.
**Recommendation.** Loud failure per section 5.7.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-API-011 — Severity: Medium
**Finding.** Behavioural guarantees not documented; consumers depend on accidental behaviour.
**Recommendation.** Explicit behaviour section per section 9.2.
**Owner.** ai-platform-eng + feature teams, sprint N+3.

### PROMPT-API-012 — Severity: Medium
**Finding.** Non-guarantees not documented; consumers depend on properties that may change.
**Recommendation.** Explicit non-guarantees per section 3.7.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-API-013 — Severity: Medium
**Finding.** Impact analysis on prompt changes is manual and inconsistent.
**Recommendation.** Process per section 7.2; automated registry-based identification.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-API-014 — Severity: Medium
**Finding.** Consumer test specificity not reviewed; some tests too strict, some too lax.
**Recommendation.** Per section 6.5 / 6.6; periodic review.
**Owner.** ai-platform-eng + consumer teams, sprint N+4.

### PROMPT-API-015 — Severity: Low
**Finding.** Deprecation announcements informal; missed by some consumers.
**Recommendation.** Standard channels per section 5.3; dashboard for active deprecations.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-API-016 — Severity: Low
**Finding.** Migration support not consistent; some authors help consumers migrate, others don't.
**Recommendation.** Standard support per section 5.4; office hours during deprecation.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-API-017 — Severity: Low
**Finding.** Contract registry not searchable; consumer discovery for impact analysis manual.
**Recommendation.** Searchable registry per section 7.1.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-API-018 — Severity: Low
**Finding.** Post-deprecation reviews not held; recurring issues not addressed.
**Recommendation.** Review cadence per section 5; learnings into the lifecycle.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team building contract discipline:

- [ ] **Sprint 0 — consumer audit.** Per prompt, who consumes the output?
- [ ] **Sprint 0 — contract documentation.** Start with the high-volume prompts; document inputs, outputs, behaviour.
- [ ] **Sprint 1 — consumer registry.** Centralised; updated on changes.
- [ ] **Sprint 1 — classification discipline.** PR template includes change classification.
- [ ] **Sprint 2 — first contract tests.** For the highest-risk prompt; integrate with CI.
- [ ] **Sprint 2 — deprecation lifecycle.** Standard process; communicated.
- [ ] **Sprint 3 — broader contract tests.** Per high-traffic prompt.
- [ ] **Sprint 3 — impact analysis process.** PR step that consults the registry.
- [ ] **Sprint 4 — owner assignments.** Cross-team prompts have dedicated owners.
- [ ] **Sprint 4 — chain dependency tracking.** Chained prompts in registry.
- [ ] **Ongoing — quarterly review.** Contract accuracy; consumer test quality; deprecation cadence.

For a team retrofitting:

- [ ] **Sprint 0 — surface implicit contracts.** What do consumers actually depend on? Survey.
- [ ] **Sprint 1 — registry + first docs.** Start with the most-consumed prompts.
- [ ] **Sprint 2 — contract tests for highest-risk.** Demonstrate value.
- [ ] **Sprint 3 — broader rollout.** Scale the practice.

A team that completes the sequence has prompt changes that don't surprise consumers and a clear process when they need to. A team that doesn't has recurring incidents from changes whose consumer impact wasn't understood.

---

## 13. References

- [prompts-as-code-discipline.md](./prompts-as-code-discipline.md) — prompts as artefacts; the foundation for contract treatment.
- [prompt-versioning.md](./prompt-versioning.md) — semantic versioning; the mechanism for the contract's version.
- [prompt-libraries-and-components.md](./prompt-libraries-and-components.md) — components as contracts within prompts.
- [structured-output-engineering.md](./structured-output-engineering.md) — schema as the most explicit contract layer.
- [few-shot-engineering.md](./few-shot-engineering.md) — example sets as contract elements.
- [prompt-ab-testing.md](./prompt-ab-testing.md) — A/B testing changes that may have contract implications.
- [prompt-anti-patterns.md](./prompt-anti-patterns.md) — including "prompt as text; change freely" and "no version pinning."
- [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — eval contract elements.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — gate that protects against contract regressions.
- [cicd-and-eval-gates/prompt-version-pinning.md](../cicd-and-eval-gates/prompt-version-pinning.md) — pinning enforces the contract surface.
- [agent-engineering/agent-vs-workflow-decision.md](../agent-engineering/agent-vs-workflow-decision.md) — long prompt chains may indicate workflow vs chain.
- [agent-engineering/agent-observability.md](../agent-engineering/agent-observability.md) — prompt-version metadata in traces.
- "Consumer-Driven Contracts" (Robinson, 2006) — the API contract pattern this practice adapts.
- API evolution guidance (e.g., Google's API design guide) — general API discipline applied here.
- Postel's law ("be liberal in what you accept, conservative in what you produce") — the contract perspective rooted in long-standing API design.
