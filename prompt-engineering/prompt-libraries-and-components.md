# Prompt Libraries and Components

> **Audience.** Engineers and prompt authors at a team or organisation with multiple AI features. Tech leads watching the team's prompts duplicate, diverge, and bloat across features. Platform engineers building shared prompt infrastructure. **Scope.** The engineering practice of componentising prompts — shared system-prompt fragments, reusable instruction blocks, formatter modules, the platform-level prompt library, and the per-feature overlay model. Not the prompt versioning depth (see [prompt-versioning.md](./prompt-versioning.md)). Not template-engine evaluation (covered briefly in section 5). **Worked client.** Meridian Health.

---

## 1. Why this document exists

A team's first AI feature has one prompt. The second has one prompt. After a year of building, the team has 30 prompts — many of which contain large fragments that are nearly identical. The "you are a helpful AI assistant for Meridian Health" preamble. The HIPAA-compliance reminder. The "if uncertain, escalate" disclaimer. The output-format JSON schema instructions. Each of these appears in 5–25 prompts, each lightly varied, each maintained independently by whichever team owns the host prompt.

When the legal team updates the HIPAA disclaimer, the change has to be made in 25 places. When the platform team improves the output-format instruction, 12 prompts use the old version and 8 use the new and 5 have a hybrid. The team spends ten hours per quarter reconciling these drifts, sometimes silently failing to do so until a compliance audit notices.

The componentisation discipline addresses this directly. Prompts compose from reusable parts. The HIPAA disclaimer is one component owned by one team; every prompt that includes it gets the current version automatically. The output-format instruction is one component owned by the platform team; updates propagate. Per-feature prompts contain feature-specific logic plus references to shared components.

The practice is engineering software engineering applied to prompts: the same DRY (don't repeat yourself) principle, the same dependency-and-versioning model, the same release process. The mechanics are mundane — a template engine, a component registry, version pinning — but the team-level effect is significant: changes that previously required coordinating across 20 prompts now require coordinating across 1 component.

This document is opinionated about four things:

1. **Shared fragments are extracted into components owned by their natural owners.** Legal owns disclaimers; clinical owns clinical instructions; platform owns format constraints. Each component has a single source of truth.
2. **Composition is explicit, not implicit.** Prompts declare their components; the composition is auditable; the assembled prompt is reproducible.
3. **Components are versioned.** Per [prompt-versioning.md](./prompt-versioning.md). Consumers pin component versions; updates propagate via deliberate consumer-side promotion.
4. **The platform's job is the library, not the prompts.** Platform owns infrastructure (the library, the registry, the composer, the eval); features own their prompts (which use components from the library).

Structure: (2) the case for componentisation; (3) component types and ownership; (4) the composition model; (5) template engines; (6) the platform library pattern; (7) versioning and dependency management; (8) eval and observability for composed prompts; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The case for componentisation

Why this discipline pays off.

### 2.1 The duplication problem (observed)

Audit findings from a typical mid-size AI team after 18 months:

- The HIPAA disclaimer appears in 27 prompts; 5 different versions live in production.
- The "you are an AI assistant" preamble appears in all prompts but with subtle variations (some say "AI assistant," some "AI helper," some "AI agent"); the variation has no engineering rationale.
- Output-format JSON schema instructions are copy-pasted into 18 prompts; 3 versions differ in how trailing commas are handled.
- The "if uncertain, escalate to a human" instruction is in 14 prompts with 6 different escalation paths described.

Each duplication is its own minor maintenance burden. Cumulatively, they're a significant fraction of the team's prompt-engineering effort.

### 2.2 The drift problem

When a fragment is duplicated, updates drift. A legal review last quarter required HIPAA disclaimer changes; the change landed in 18 of 27 prompts. The remaining 9 still have the old disclaimer. Nobody noticed until the next audit. The remediation is hand-editing 9 prompts and updating the eval to catch this kind of drift in future.

Componentisation prevents this. The disclaimer is one component; one update propagates everywhere.

### 2.3 The ownership problem

When fragments are duplicated, ownership is diffuse. Who owns the HIPAA disclaimer? The legal team writes it; the engineering team includes it; the prompt-owner-feature-team updates it. When the disclaimer needs to change, the conversation is "whose prompt is this?" rather than "what's the disclaimer say?"

Components have explicit owners. The HIPAA disclaimer component's owner is the legal team. The legal team updates the component; consumers receive the update.

### 2.4 The cost problem

Identical content paid for at input token rates × multiple prompts × call volume. Each duplicated paragraph costs.

Componentisation doesn't directly reduce per-call cost (the assembled prompt still includes the component's tokens), but it reduces the cost of *maintaining* the prompts — engineering time on duplication is significant.

(Prompt caching, per [caching-for-cost.md](../cost-and-finops/caching-for-cost.md), does reduce per-call cost for stable shared content — a reason to extract common fragments specifically.)

### 2.5 The quality problem

Duplicate fragments often diverge in quality. The team has 27 versions of the disclaimer; some are better-worded than others. Componentisation centralises improvement: one well-crafted version is used everywhere.

### 2.6 The scope criterion

Componentisation pays off when:

- 3+ prompts share content.
- The content has a natural owner (one team responsible for what it says).
- The content changes occasionally (so the propagation matters).
- The team has at least one technical owner for the prompt library.

Below this threshold, the overhead of components may not be worth it; treat the duplication as acceptable.

---

## 3. Component types and ownership

Different kinds of content warrant different component types.

### 3.1 The component taxonomy

| Type | Example | Typical owner |
| --- | --- | --- |
| Preamble | "You are an AI assistant for Meridian Health..." | Platform |
| Disclaimer | HIPAA, terms of use, refusal-to-give-medical-advice | Legal / Compliance |
| Domain-context | "Meridian is a healthcare technology platform serving..." | Domain team |
| Format instruction | "Output as JSON with fields: X, Y, Z..." | Per-feature or platform |
| Behaviour instruction | "When uncertain, escalate to..." | Per-feature or platform |
| Tool catalogue documentation | The system-prompt fragment describing tools | Platform |
| Style instruction | "Use clinical tone; avoid hedging" | Per-feature or domain |
| Refusal pattern | "If asked to do X, respond..." | Safety / Compliance |
| Example block | A few-shot example set | Per-feature (per [few-shot-engineering.md](./few-shot-engineering.md)) |
| Citation format | "Cite sources as [DOC-ID, section]" | Platform |

Each type has a natural owner and a natural cadence of change.

### 3.2 Ownership matters

The owner is the source of truth for the component. They:

- Write the canonical version.
- Approve changes.
- Coordinate with consumers when breaking changes are needed.
- Maintain the eval that protects the component's correctness.

Without clear ownership, components devolve back into "everyone's responsibility, no one's" — same problem componentisation was supposed to solve, now at the component level.

### 3.3 The owner's responsibilities

For each component:

- **Specification.** What this component does, what it's for, what its constraints are.
- **The text.** The current canonical version, versioned.
- **The eval.** Cases that verify the component does what it's supposed to do.
- **Change process.** PR template, review requirements, eval requirements before promotion.
- **Consumers.** A list (or query) of which prompts use this component.

### 3.4 Consumer-driven design

Components are designed for consumers' needs, not for the component author's preferences. The HIPAA disclaimer's text matters because consumers depend on it; changes have to consider consumer impact. The component owner asks consumers before making consumer-affecting changes.

This is the same discipline as API design. The component is an API; consumers depend on it; backward compatibility matters.

### 3.5 Cross-team components

Some components span teams. The "Meridian context" is written by the domain team, reviewed by the platform team, used by all feature teams. The cross-team ownership is documented; the change process accommodates multiple stakeholders.

The friction of cross-team components is real. Reserve cross-team components for content that genuinely benefits from cross-team coordination; don't force componentisation when team-specific components would work fine.

### 3.6 Component vs prompt boundary

When is a fragment a "component" vs just part of a prompt? Heuristics:

- **3+ prompts use it (or will).** Single-use content stays in the prompt.
- **The content has its own change cadence.** The HIPAA disclaimer changes when legal updates; the prompt's behaviour instructions change with the feature.
- **The owner differs from the prompt's owner.** Legal owns the disclaimer; the feature team owns the prompt.

If none apply, leave it inline. Componentisation overhead doesn't pay off for content that's truly prompt-specific.

---

## 4. The composition model

How components combine into a final prompt.

### 4.1 Explicit composition

The prompt declares its components:

```yaml
prompt: care-coordinator-system
version: v23
components:
  - id: platform-preamble
    version: v3
  - id: meridian-domain-context
    version: v4
  - id: hipaa-disclaimer
    version: v7
  - id: care-coordinator-behaviour
    version: v12
  - id: care-coordinator-tools-catalogue
    version: v8
  - id: citation-format
    version: v2
  - id: escalation-instruction
    version: v5
inline_content: |
  [prompt-specific content here]
```

The components are pinned by version. Updates are deliberate (the consumer changes the version pin).

### 4.2 The assembly process

A composer service / library reads the declaration; assembles the components in order; appends the inline content; returns the final prompt string. Deterministic.

```python
def assemble_prompt(prompt_spec):
    parts = []
    for component_ref in prompt_spec.components:
        component = component_registry.get(component_ref.id, component_ref.version)
        parts.append(component.text)
    if prompt_spec.inline_content:
        parts.append(prompt_spec.inline_content)
    return "\n\n".join(parts)
```

The assembled prompt is what reaches the model. The trace records the prompt's specification (which components, which versions), not just the assembled text.

### 4.3 The composition order

Components are ordered intentionally. Typical orderings:

1. Platform preamble (broad context).
2. Domain context (Meridian-specific).
3. Disclaimers (compliance).
4. Feature-specific behaviour.
5. Tool catalogue (if agent).
6. Format / style instructions.
7. Examples (if any).
8. The user's input (inserted at runtime).

The ordering reflects how the model reads context: broad-to-specific, with the user's input last for maximum attention.

### 4.4 Conditional composition

Sometimes components are conditionally included:

- HIPAA disclaimer only for healthcare contexts.
- Tool catalogue only for agent contexts.
- Specific examples only for specific tenant tiers.

The composer supports conditions:

```yaml
components:
  - id: hipaa-disclaimer
    condition: "feature.domain == 'healthcare'"
    version: v7
```

Conditional logic should be sparing; complex conditions make the assembled prompt unpredictable. Keep conditions simple and well-tested.

### 4.5 Per-component parameters

Some components are parameterised:

```yaml
- id: citation-format
  version: v2
  params:
    citation_style: "bracket"  # or "footnote" or "inline"
```

The component's text uses the parameter:

```
Cite sources using {{ citation_style }} style: ...
```

Parameters are useful but should be limited. A component with many parameters is a sign it's actually multiple components in disguise; split it.

### 4.6 The "what if a component is missing" failure

If the composer can't find a referenced component (e.g., the registry doesn't have the version pinned), it must fail loudly. Silent fallback to a default would produce a different prompt than intended; the failure must surface so the team fixes the reference.

Discipline: in CI, the prompt's composition is validated; missing components fail the build. In production, the composer's request to the registry has timeouts and retries; failure produces an alert.

### 4.7 Caching the assembled prompt

For high-volume features, assembling the prompt on every call is wasteful. Cache:

- Key: the prompt's specification (components + versions + inline content + parameters).
- Value: the assembled prompt text.
- TTL: hours (the cache flushes on prompt version updates; the version captures component versions).

The cache reduces composition cost. For frontier-model prompt caching (per [caching-for-cost.md](../cost-and-finops/caching-for-cost.md)), the stable system-prompt portion (composed of stable components) is the natural cache prefix.

---

## 5. Template engines

The mechanical layer for composition.

### 5.1 Common options

Most teams use a generic template engine adapted for prompts:

- **Jinja2 (Python).** Mature; widely used.
- **Handlebars / Mustache.** Logic-less; predictable.
- **Custom DSL.** Some teams build a thin prompt-specific layer over Jinja or text concatenation.

The engine choice is tactical. The discipline (componentisation, versioning, eval) is engine-agnostic.

### 5.2 What the engine needs to do

- Concatenate component texts with appropriate separators.
- Substitute parameters.
- Evaluate simple conditions.
- Produce a deterministic output given the same inputs.

What it doesn't need to do:

- Complex logic.
- I/O during rendering.
- Side effects.

Logic should be in the composer (Python code), not the template (text). The template is text + minimal substitution.

### 5.3 The "template logic gets too complex" failure

Some teams try to do too much in templates: nested conditions, loops, complex branching. The templates become hard to read and harder to debug. The corrective: move logic to code; keep templates simple.

A template with more than 3 levels of nested logic is a smell. Refactor.

### 5.4 Multi-language support

If the team's components are used by services in multiple languages (Python, TypeScript, Go), the composer needs to produce the same output in each. Options:

- A single language for the composer (gateway service), called from any language.
- Multiple composer implementations with strict cross-language conformance tests.
- A pre-assembly step in CI that produces the final prompts as static artefacts; services consume the static prompts.

The single-composer-service approach is cleanest at scale.

### 5.5 The composed-prompt artefact

Some teams treat composed prompts as build artefacts:

- CI assembles all prompts after every prompt or component change.
- The assembled prompts are versioned and stored in the prompt store.
- Runtime services fetch the assembled prompt by version; no composition at request time.

The pattern simplifies the runtime, complicates the build. Worth it for high-scale deployments.

---

## 6. The platform library pattern

When the team is mature enough, the platform team owns a prompt library shared across feature teams.

### 6.1 What the platform library contains

- Common preambles.
- Format-instruction patterns (output-as-JSON, output-as-markdown, output-as-citation-list).
- Tool-catalogue documentation patterns.
- Escalation patterns.
- Style and tone components for common voices.
- Refusal patterns.
- Disclaimer and compliance components (in coordination with legal / compliance).

The library is the team's "off-the-shelf" prompt parts; features compose from them rather than re-deriving.

### 6.2 The library's API

```
GET /components/{id}/versions
GET /components/{id}/{version}
POST /prompts/assemble
```

Features call the library's composer to assemble their prompts. The library returns assembled prompts; features consume.

### 6.3 The library's eval

The library has its own eval surface:

- Per-component eval: does the component do what it's supposed to do?
- Cross-component eval: do components compose without conflicts?
- Consumer eval: do consumer prompts that depend on a component still pass eval after a component change?

Library changes go through:

- Library's own eval (per-component, cross-component).
- Consumer eval (downstream features that pin the component's new version).
- Promotion through the eval gate per [eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md).

### 6.4 The library's release cadence

Library components release on their own cadence — typically slower than feature prompts. A library component release:

- Component PR with eval delta.
- Library team review.
- Eval gate pass.
- Tagged release; documented in changelog.
- Consumers notified; consumers pin the new version at their own pace.

Forced-version-bumps are rare (and only for safety/compliance reasons); consumer-driven adoption is the norm.

### 6.5 The library team

The library has owners. The library team typically:

- 1–3 platform engineers.
- Coordinates with feature teams.
- Liaises with legal, compliance, security for relevant components.
- Maintains the library's documentation, eval, and registry.
- Reviews library PRs.

A library without owners atrophies. Investing in the library is investing in the team's prompt-engineering velocity.

### 6.6 Library success metrics

- **Component reuse rate.** How many prompts use each component? Components that are underused signal either bad design or bad discoverability.
- **Time to component update propagation.** How long after a component update do consumers adopt the new version? Slow propagation indicates friction.
- **Component-driven incidents.** How many incidents were caused by component bugs? Should be low; near-zero is ideal.

Tracked quarterly; informs library team priorities.

### 6.7 When NOT to centralise

Small teams (1–3 engineers, 1–3 features) often don't benefit from a platform library. The overhead exceeds the benefit. Per-team components or inline content is fine.

The library pattern fits teams with 5+ AI features across 3+ teams. Below that, the discipline is "be aware of duplication and componentise opportunistically."

---

## 7. Versioning and dependency management

Components are versioned artefacts; dependencies are explicit.

### 7.1 Semantic versioning for components

Per [prompt-versioning.md](./prompt-versioning.md), components follow semantic versioning:

- **Major (X.0.0).** Breaking changes to the component's behaviour or output shape.
- **Minor (X.Y.0).** Backward-compatible enhancements.
- **Patch (X.Y.Z).** Backward-compatible fixes.

Major bumps require consumer migration. Minor and patch don't (but consumers should still eval).

### 7.2 Version pinning

Consumers pin component versions:

```yaml
components:
  - id: hipaa-disclaimer
    version: ~7.2  # use 7.2.x, accept patches
  - id: platform-preamble
    version: =3.0.0  # exact
  - id: citation-format
    version: ^2.1  # use 2.x.x starting from 2.1, accept minor and patch
```

Pin precision is per consumer's risk tolerance.

### 7.3 Dependency resolution

When a prompt declares components, the composer resolves the versions:

- Exact pins are used as-is.
- Range pins resolve to the latest matching version in the registry at compose time.
- Resolution result is recorded in the assembled prompt's metadata; same input → same output.

### 7.4 The "transitively shared component" pattern

Some components reference other components. The platform preamble might reference the standard tone instruction. The composer resolves transitively.

Cycles must be detected and rejected. The registry validates that no component creates a circular dependency.

### 7.5 Update propagation

When a component releases a new version:

- The library team announces (changelog, team chat).
- Consumers update their version pins (PR per consumer).
- Each consumer's eval runs against the new version.
- Consumer prompts promote on eval pass.

The cadence is consumer-driven. The library team can recommend; only the consumer can promote.

### 7.6 Forced updates

For safety/compliance components, the library team may need to force-update consumers (e.g., a HIPAA disclaimer fix that must propagate immediately). The pattern:

- The library team flags the version as "critical update."
- Consumer prompts are blocked from new releases until they pin the critical version.
- After a deadline, the prior version is removed from the registry; older pins fail to compose.

Use sparingly. Forced updates strain consumer relationships.

### 7.7 Component deprecation

Components are retired when no longer needed:

1. Mark the component deprecated.
2. Notify consumers; suggest replacement.
3. Wait for consumer migration.
4. Remove from active registry; preserve in archive.

The lifecycle mirrors API deprecation. Same discipline.

---

## 8. Eval and observability for composed prompts

The eval and observability surfaces for componentised prompts.

### 8.1 Per-component eval

Per section 6.3. Each component has its own eval:

- Component-specific golden set (inputs that exercise the component's purpose).
- Outcome eval (does the component produce its intended effect?).
- Behaviour-stability eval (does the component still behave as expected after edits?).

### 8.2 Per-prompt eval

Standard prompt eval (per [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md)) runs on the assembled prompt. The composition is invisible to the prompt's eval — the prompt is what reaches the model.

Component updates can cause prompt-eval regressions; the consumer's eval catches them.

### 8.3 Cross-component eval

Components can conflict. The platform preamble says "be concise"; the citation format component says "always include full citation details." The combination produces verbose output despite the preamble's instruction.

Cross-component eval tests the combinations:

- Library publishes "compatibility tests" — assembled prompts using common component combinations.
- Tests verify combinations produce expected behaviour.
- Conflicts identified and resolved (often by clarifying component scope).

### 8.4 Observability — what's in the trace

Per LLM call:

- The prompt's specification (components, versions).
- The assembled prompt's hash (for deterministic identification).
- Component-level attribution.

The trace shows: "this call used hipaa-disclaimer v7.2.1, platform-preamble v3.0.0, care-coordinator-behaviour v12.4.0, ..." Useful for debugging "why did the agent respond differently today?" — compare component versions across periods.

### 8.5 Component impact analysis

A library team query: "for component X v6 vs v7, what was the per-consumer eval delta?" Aggregates per-consumer eval data; reports the component change's net impact.

Used for understanding which component changes had broad positive impact vs negative. Informs the library team's roadmap.

### 8.6 Continuous eval

Per [agent-evals.md](../agent-engineering/agent-evals.md), continuous eval over production traces catches drift. For componentised prompts, the drift signals the component or the prompt; per-component tagging in the trace helps localise.

---

## 9. Worked Meridian example

Meridian's prompt library, in production for ~10 months.

### 9.1 Library scope

The platform team owns `meridian-prompt-library` with ~25 components:

- **Platform components (8):** preamble, format-instructions (JSON, markdown, plain), citation-format, escalation, tool-catalogue-template, structured-output-schema, default-style.
- **Compliance components (4):** hipaa-disclaimer, terms-of-use, refusal-medical-advice, refusal-out-of-scope.
- **Domain components (6):** meridian-context, clinical-tone, payer-tone, developer-tone, care-coordinator-domain, patient-summary-domain.
- **Specialised components (7):** clinical-note-formatter, soap-format, soap-with-citations, sql-generation-style, analytics-narrative-tone, accessibility-style, multi-language-disclaimer.

Each has an owner (platform / legal / clinical / analytics).

### 9.2 Composition

The care-coordinator's prompt:

```yaml
prompt: care-coordinator-system
version: v23
components:
  - id: meridian-platform-preamble
    version: =3.0.0
  - id: meridian-context
    version: ^4.1
  - id: hipaa-disclaimer
    version: =7.2.1
  - id: clinical-tone
    version: ^2.0
  - id: care-coordinator-domain
    version: ^5.3
  - id: care-coordinator-tools-catalogue
    version: ^8.1
  - id: citation-format
    version: ^2.0
  - id: escalation
    version: ^5.0
inline_content: |
  [Care-coordinator-specific behavior instructions]
  [Reasoning patterns specific to care coordination]
  [Few-shot examples — see registry]
```

The inline content is ~600 tokens; the composed components add ~1800 tokens; total system prompt is ~2400 tokens.

### 9.3 Component update propagation

In 10 months, ~40 component releases. The cadence varies:

- HIPAA disclaimer: 2 updates (one legal-driven, one wording improvement).
- Platform preamble: 1 update (major; v2 → v3 with broader tone change).
- Tool-catalogue-template: 6 updates (responds to tool architecture evolution).
- Citation format: 3 updates (incremental improvements; analyst feedback).
- Clinical tone: 4 updates (clinical team refinements).

Consumer adoption typically 1–2 weeks behind the library release; faster when the update is critical, slower when it's enhancement.

### 9.4 The library team

- 1 senior platform engineer (~30% of time).
- 1 platform engineer (~20% of time).
- Cross-functional reviewers: clinical-content team, legal, security.

Total ~0.5 FTE on the library. Library team's deliverables: component PRs, consumer guidance, eval coverage of components, quarterly library health report.

### 9.5 Effects observed

After 10 months:

- HIPAA disclaimer changes now propagate in < 1 week (was ~4 weeks pre-library).
- Cross-feature consistency is high (the team rarely finds prompts saying different things about the same topic).
- Per-feature prompt sizes are smaller (the inline-content portion; the assembled portion is similar).
- Library team coordinates with legal/compliance directly; feature teams aren't the intermediary.

### 9.6 What didn't work initially

- **Too many components.** Early version had ~50 components; many were under-used. Trimmed to ~25 by combining or removing.
- **Over-parameterised components.** A few components had many parameters; they became hard to use and harder to eval. Split into multiple specific components.
- **Resistance from feature teams.** "We want to write our own prompts." Resolved by demonstrating the time savings on the next compliance-driven update.
- **No eval initially.** Components shipped without component-level eval; the first regression caught by a downstream consumer's eval. Added component-level eval after.

### 9.7 The discipline that worked

- **Strict ownership.** Each component has one owner; no committee.
- **Consumer-driven version pinning.** Consumers control their adoption pace; library team doesn't force.
- **Eval at component AND prompt levels.** Both surfaces; catch issues at the right layer.
- **Documented composition order.** Components have intended position in the composition; documented; enforced by the composer.

### 9.8 The fine-tune interaction

The patient-summary fine-tuned model (per [few-shot-engineering.md](./few-shot-engineering.md) section 9.5) is fine-tuned on prompts that include the relevant components. When components update, the fine-tune doesn't automatically benefit; the re-fine-tune (quarterly) picks up the latest components.

The team accepts the lag. The components that matter most for fine-tuned features change rarely.

---

## 10. Anti-patterns

### 10.1 "Inline strings duplicated across prompts"

Each prompt has its own copy of the HIPAA disclaimer, the preamble, the format instruction. Updates require multi-file edits.

**Corrective.** Extract to components per section 3.

### 10.2 "Components without owners"

Components exist but no one is responsible for any specific one. Updates stall; quality degrades.

**Corrective.** Per-component owner per section 3.2.

### 10.3 "Components without versions"

Components are mutable; updates take effect immediately for all consumers; consumers can't pin.

**Corrective.** Semantic versioning per section 7.

### 10.4 "Over-componentisation"

Every paragraph is a component; the prompt is mostly composition references. Eval is hard because nothing is together; composition cost is high.

**Corrective.** Componentise content meeting the section 3.6 criteria; leave the rest inline.

### 10.5 "Component logic too complex"

Components have nested conditionals, loops, multiple parameters. Hard to understand; hard to debug.

**Corrective.** Simple components; complex logic in the composer (code), not in the template.

### 10.6 "No cross-component eval"

Components are eval'd individually; combinations not tested; conflicts emerge in production.

**Corrective.** Cross-component eval per section 8.3.

### 10.7 "Forced updates as default"

Library team forces consumers to update with every release. Consumer relationships strained.

**Corrective.** Consumer-driven adoption per section 7.5; forced updates only for safety/compliance per section 7.6.

### 10.8 "No library when one is needed"

Team has 15+ prompts with significant duplication; never invests in componentisation; same fragments live in many places.

**Corrective.** Library investment per section 6 when criteria (section 2.6) are met.

---

## 11. Findings (sprint-assignable)

### PROMPT-LIB-001 — Severity: Critical
**Finding.** Compliance content (HIPAA disclaimer, refusal patterns) duplicated across 15+ prompts; updates propagate slowly and inconsistently.
**Recommendation.** Extract to components per section 3; assign legal/compliance as owners; consumer migration.
**Owner.** ai-platform-eng + legal, sprint N+1.

### PROMPT-LIB-002 — Severity: High
**Finding.** Prompt fragments duplicated significantly across features; estimated 30%+ of prompt content is repeated.
**Recommendation.** Audit per section 2.1; extract top duplicated fragments to components.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-LIB-003 — Severity: High
**Finding.** Components exist but lack ownership; updates stall waiting for "someone" to act.
**Recommendation.** Per-component owner per section 3.2; documented; reviewed quarterly.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-LIB-004 — Severity: High
**Finding.** Components not versioned; updates take effect for all consumers immediately; consumer prompts regress silently.
**Recommendation.** Semantic versioning per section 7; consumer pinning per section 7.2.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-LIB-005 — Severity: High
**Finding.** Composer doesn't fail on missing components; default fallback masks errors.
**Recommendation.** Loud failure per section 4.6; CI validation; production alerting.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-LIB-006 — Severity: Medium
**Finding.** Component-level eval absent; component changes promoted without quality verification.
**Recommendation.** Per-component eval per section 6.3 / 8.1.
**Owner.** ai-platform-eng + feature teams, sprint N+3.

### PROMPT-LIB-007 — Severity: Medium
**Finding.** Cross-component conflicts emerge in production; combinations not tested pre-promotion.
**Recommendation.** Cross-component eval per section 8.3.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-LIB-008 — Severity: Medium
**Finding.** Component update propagation slow; consumers months behind library.
**Recommendation.** Propagation metric per section 6.6; quarterly review; reduce friction.
**Owner.** ai-platform-eng + feature teams, sprint N+3.

### PROMPT-LIB-009 — Severity: Medium
**Finding.** Library has > 50 components; many under-used; discoverability poor.
**Recommendation.** Audit per section 9.6; consolidate or retire under-used components.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-LIB-010 — Severity: Medium
**Finding.** Assembled prompt not visible in trace; debugging requires manual composition.
**Recommendation.** Composition metadata in trace per section 8.4; including component versions.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-LIB-011 — Severity: Medium
**Finding.** Template engine logic complex; templates hard to read and debug.
**Recommendation.** Simplify per section 5.3; move logic to composer code.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-LIB-012 — Severity: Medium
**Finding.** Components with many parameters; should be split into specific components.
**Recommendation.** Per section 4.5; component-per-purpose; split parameter-heavy components.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-LIB-013 — Severity: Medium
**Finding.** Component deprecation lifecycle absent; deprecated components linger.
**Recommendation.** Lifecycle per section 7.7; archive process.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-LIB-014 — Severity: Medium
**Finding.** Library team is undersized; component updates queue.
**Recommendation.** Library team sizing per section 6.5; ~0.5 FTE minimum.
**Owner.** ai-platform-eng + leadership, sprint N+4.

### PROMPT-LIB-015 — Severity: Low
**Finding.** No platform-library documentation; new feature teams unaware of components.
**Recommendation.** Library docs site; discoverable; updated with each release.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-LIB-016 — Severity: Low
**Finding.** Multi-language composer not implemented; teams in different languages re-derive composition logic.
**Recommendation.** Single composer service per section 5.4.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-LIB-017 — Severity: Low
**Finding.** Component changelog not maintained; consumers can't tell what changed.
**Recommendation.** Per-component changelog; required at release.
**Owner.** ai-platform-eng + library team, sprint N+5.

### PROMPT-LIB-018 — Severity: Low
**Finding.** Component success metrics not tracked; library team flying blind on impact.
**Recommendation.** Metrics per section 6.6; quarterly library health report.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team building componentisation:

- [ ] **Sprint 0 — duplication audit.** What content is shared across prompts? Quantify duplication.
- [ ] **Sprint 0 — owner identification.** For each candidate component, who would own it?
- [ ] **Sprint 1 — first extraction.** Extract 2–3 highest-value components (often compliance content); assign owners.
- [ ] **Sprint 1 — composer.** Build (or adopt) a template engine + composer.
- [ ] **Sprint 1 — component versioning.** Semantic versioning; consumer pinning.
- [ ] **Sprint 2 — consumer migration.** Migrate prompts that use the extracted components.
- [ ] **Sprint 2 — eval.** Per-component eval; cross-component eval.
- [ ] **Sprint 2 — observability.** Composition metadata in traces.
- [ ] **Sprint 3 — expand library.** Extract more components as duplication emerges.
- [ ] **Sprint 3 — library team.** Formal ownership; documented charter; sizing.
- [ ] **Sprint 4 — propagation process.** Update notification; consumer adoption tracking.
- [ ] **Ongoing — quarterly library review.** Health metrics; audit; trim under-used.

For a team with an existing under-componentised state:

- [ ] **Sprint 0 — duplication audit.** Surface the current state; quantify the cost.
- [ ] **Sprint 1 — high-impact extractions.** Pick the 3 most-duplicated fragments; extract; demonstrate value.
- [ ] **Sprint 2 — library investment decision.** Based on demonstrated value; size the library team.
- [ ] **Sprint 3 — broader extraction.** Build on the pattern.

A team that completes the sequence has prompts that compose from a library; updates propagate cleanly; cross-feature consistency is high. A team that doesn't has a maintenance burden that grows linearly with prompt count and updates that take weeks.

---

## 13. References

- [prompts-as-code-discipline.md](./prompts-as-code-discipline.md) — broader prompt-as-code discipline; components inherit it.
- [prompt-versioning.md](./prompt-versioning.md) — semantic versioning applied to components.
- [few-shot-engineering.md](./few-shot-engineering.md) — example sets as reusable components.
- [structured-output-engineering.md](./structured-output-engineering.md) — format components.
- [prompt-ab-testing.md](./prompt-ab-testing.md) — A/B testing component changes.
- [prompt-as-api-contract.md](./prompt-as-api-contract.md) — component-as-contract perspective.
- [prompt-anti-patterns.md](./prompt-anti-patterns.md) — duplication and inline-string anti-patterns.
- [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — eval practice that covers component and prompt-level eval.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — gate that protects component and prompt promotions.
- [cicd-and-eval-gates/prompt-version-pinning.md](../cicd-and-eval-gates/prompt-version-pinning.md) — release-side pinning of prompt and component versions.
- [agent-engineering/agent-observability.md](../agent-engineering/agent-observability.md) — trace metadata that includes composition.
- [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — span attributes that capture prompt composition.
- [cost-and-finops/caching-for-cost.md](../cost-and-finops/caching-for-cost.md) — prompt caching naturally aligns with stable shared components.
- Jinja2, Handlebars, Mustache — common template engines adapted for prompts.
- Software-engineering DRY principle — the broader discipline this practice applies.
