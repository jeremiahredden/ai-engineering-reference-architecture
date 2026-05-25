# Golden Set Design

> **Audience.** Engineers and tech leads building or refactoring the golden set for an AI feature. Anyone whose eval suite is "20 cases that grew organically and nobody is sure what they cover." **Scope.** The *engineering* practice of designing, curating, and growing a golden set. Pair with [eval-engineering-playbook.md](./eval-engineering-playbook.md) (the broader eval practice), [llm-as-judge-patterns.md](./llm-as-judge-patterns.md) (the judge that scores against the golden set), and [regression-eval-suites.md](./regression-eval-suites.md) (the regression suite that grows from production bugs). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The golden set is the load-bearing artifact of the eval practice. Every other piece — the judge, the eval gate, the dashboards, the regression discipline — is engineering around the golden set. A team with a thoughtful golden set has an eval practice; a team without one has dashboards.

Most teams' golden sets accumulate organically. A few cases get added during initial development, more get added during dogfooding, a handful from bug reports. Eighteen months later the suite has 80 cases nobody can confidently describe — which classes are covered, which are not, which were once important and are now obsolete, which were added by an engineer who left two years ago.

This document is the discipline for golden-set design that survives that drift. The discipline is not "have more cases"; it is *deliberate case selection*, *explicit class coverage*, *clear ownership*, and *engineered refresh*. A 200-case golden set built with discipline is more useful than a 2,000-case suite built by accumulation.

This document is opinionated about three things:

1. **Coverage is engineered, not accumulated.** The golden set covers known case classes deliberately. Adding a case is a decision; the team can articulate which class it covers and why it earned a slot.
2. **Every case has an owner.** A team or person accountable for the case's continued relevance. Orphan cases (added by people no longer on the team) decay; the discipline detects and retires them.
3. **The golden set is sized for the practice, not for impressive numbers.** A 100-case suite that is human-reviewable in two days is more useful than a 5,000-case suite the team cannot recheck.

Structure: (2) the case selection framework; (3) the case-class taxonomy; (4) case structure and metadata; (5) the scoring rubric; (6) ownership and refresh discipline; (7) growth patterns; (8) integration with the broader eval practice; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The case selection framework

A golden set is a sample of the workload's case space. Like any sample, what you put in it determines what you measure.

### 2.1 The four selection criteria

A balanced golden set selects cases against four criteria. Each accounts for ~25% of the suite at maturity.

**Representative.** The most common shape of user interaction. If 60% of traffic is lookup-shaped questions, ~25% of the golden set is lookup-shaped questions. The representative cases ensure the eval measures performance on the bulk of production.

**Edge.** Adversarial inputs (ambiguous, missing context, contradictory premise), uncommon-but-important shapes, the long tail. Edge cases ensure the eval catches degradation on the cases that production handles less often.

**Known-failure.** Real cases that have produced wrong answers — in dogfooding, in pilots, in early production. These cases test the system against its known weaknesses; regressions here are detected immediately.

**High-stakes.** The cases where a wrong answer is most expensive — regulated content, customer-visible commitments, clinical decisions. These cases test the system against the workload's reputation-critical surface.

### 2.2 Why the balance

A golden set that is 100% representative misses edge cases and high-stakes cases (overconfident on bulk; blind to long tail).

A golden set that is 100% edge cases produces eval results that do not predict bulk-traffic quality (pessimistic on what matters most).

A golden set that is 100% known-failure cases becomes a regression suite (useful but different — see [regression-eval-suites.md](./regression-eval-suites.md)).

A golden set that is 100% high-stakes cases over-indexes on a narrow band of behavior.

The balance is the point. The 25/25/25/25 ratio is a starting heuristic; mature suites adjust based on workload.

### 2.3 The minimum viable golden set

Per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 3.1, the minimum viable golden set is 20 cases:

- 5 representative
- 5 edge
- 5 known-failure (or designed by domain experts if production data is unavailable)
- 5 high-stakes

20 cases is small enough to be reviewable manually every week, large enough to produce statistically-meaningful pass-rate trends, balanced enough to cover the four selection criteria.

### 2.4 The growth target

The 20-case starting point grows to 100-300 cases within a quarter and 500-1000 cases within a year. The growth is structured (per section 7); it does not just accumulate.

For Meridian Care Coordinator: 200 clinical golden cases at production maturity.

### 2.5 The size ceiling

There is a practical ceiling. A 5,000-case eval suite:
- Takes too long to run for fast feedback.
- Costs too much for routine execution.
- Is too large for any human to maintain coverage understanding.

Most teams' workloads are well-served at 200-500 cases. Beyond that, additional cases produce diminishing returns; the marginal coverage is small.

The exception: workloads with genuinely-diverse case spaces (general-purpose chatbots, broad QA systems) may justify larger suites, but they pay the operational cost.

---

## 3. The case-class taxonomy

Cases are tagged with classes. The taxonomy supports per-class analysis: which classes are passing, which are failing, where the team should invest.

### 3.1 The taxonomy dimensions

A case is described by multiple class dimensions:

**Question type.** Lookup / definition / multi-hop / comparison / synthesis / refusal / clarification.

**Domain.** For domain-specific workloads: clinical / drug-interaction / scheduling / coordination.

**Complexity.** Simple (single-step) / multi-step / multi-hop / agent-loop-required.

**Stakes.** Low / medium / high / critical.

**Production frequency.** Common / occasional / rare.

**Failure history.** Never-failed / fixed-once / chronic.

### 3.2 The case as multi-tagged

A single case carries multiple tags. For example:

```yaml
case_id: GOLD-0014
question: "What is the post-discharge follow-up for a CHF patient on the new pathway?"
class_tags:
  question_type: lookup
  domain: clinical
  complexity: simple
  stakes: high
  production_frequency: common
  failure_history: fixed_once
```

The tags support filtering and aggregation: "show pass-rate for clinical lookup cases" / "show high-stakes case pass-rate trend" / "show cases that have failed historically and their current pass-rate."

### 3.3 The taxonomy evolution

The taxonomy is itself an artifact. As the workload evolves, the taxonomy may need new dimensions:

- A new feature introduces a new question type → add the type to the taxonomy.
- A new domain area is added → add the domain.

Taxonomy changes are PR-reviewed; new tags are added with definitions.

### 3.4 The taxonomy decay

Tags that no longer correspond to real workload classes should be retired. The discipline: quarterly review of tag usage; tags used by < 5 cases are candidates for consolidation or retirement.

---

## 4. Case structure and metadata

A case is a structured artifact, not just a question-answer pair.

### 4.1 The minimum case structure

```yaml
case_id: GOLD-0014
question: "What is the post-discharge follow-up protocol for a CHF patient on the new pathway?"
expected_answer: |
  The post-discharge follow-up is a 7-day nursing check-in and a 14-day clinician
  visit. Citations: AHA 2024 HF Discharge Bundle section 3.2; Mercy Cleveland
  Protocol HF-22.

scoring:
  must_include:
    - "7-day nursing check-in"
    - "14-day clinician visit"
  must_cite:
    - "AHA 2024 HF Discharge Bundle"
    - "Mercy Cleveland Protocol HF-22"

class_tags:
  question_type: lookup
  domain: clinical
  complexity: simple
  stakes: high

source:
  added_date: 2026-03-15
  added_by: clinical-knowledge-engineering
  reason: representative_clinical_lookup
```

### 4.2 The extended case structure

For RAG cases (per [eval-of-rag.md](./eval-of-rag.md)), add:

```yaml
expected_sources:
  - chunk_id: "clinical-guideline:aha-hf-2024:section-3.2:chunk-0042"
    is_required: true
  - chunk_id: "tenant-protocol:mercy-cleveland:hf-22:chunk-0007"
    is_required: true

required_claims:
  - claim: "7-day nursing check-in is required"
    must_cite: "clinical-guideline:aha-hf-2024:section-3.2"
```

For agent cases (per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 11), add:

```yaml
trajectory_expectations:
  - tool_call: "retrieve_clinical_guideline"
    expected_args_match: "post-discharge.*CHF"
  - tool_call: "draft_followup_reminder"
    expected_args_match_patient: true
budgets:
  expected_turns_max: 3
  expected_cost_max_usd: 0.30
```

For conversational cases, the case is a sequence of turns:

```yaml
turns:
  - turn: 1
    question: "What's the post-discharge follow-up for CHF?"
    expected_answer_pattern: must_include_basic_followup
  - turn: 2
    question: "What about for elderly patients specifically?"
    expected_answer_pattern: must_include_elderly_consideration
    expected_uses_prior_context: true
```

### 4.3 The expected_answer field

The expected_answer is a *semantic reference*, not a string-match target. The judge is given the expected_answer for context; the judge decides whether the produced answer is equivalent.

Some cases use freeform expected_answer (a model-like paragraph); some use structured assertions (must_include, must_cite, must_not_include). Both shapes are supported.

The rule: the expected_answer should be unambiguous enough that a domain reviewer can confirm "yes, this answer is right" or "no, this answer is wrong" without further information.

### 4.4 The case as code

Cases are committed to version control as code (YAML, JSON, or whatever the eval framework consumes). PR-reviewed; auditable history.

For Meridian: cases live in `meridian-eval/cases/clinical/` and `meridian-eval/cases/drug-interaction/` etc. The directory structure mirrors the case-class taxonomy.

### 4.5 The case ID convention

Case IDs are stable. A case is identified by its ID throughout its lifecycle (added, updated, retired). The ID convention:

- Sequential: `GOLD-0001`, `GOLD-0002`, etc.
- Or class-prefixed: `CLIN-0042`, `DRUG-0007`, etc.

Either pattern works. Stability matters more than the specific convention.

---

## 5. The scoring rubric

The rubric defines what "pass" means.

### 5.1 The binary vs scalar choice

**Binary pass/fail.** A case passes if all criteria are met; fails otherwise. Easy to aggregate; clear thresholds; no ambiguity.

**Scalar scoring.** Each case gets a 0-1 score across multiple dimensions; aggregate is a weighted average. More nuanced; harder to act on; thresholds become fuzzy.

For most teams: start with binary. Move to scalar only when binary is genuinely insufficient (typically: when partial credit is meaningful and the team has the discipline to set thresholds for the scalar metric).

For Meridian: binary pass/fail. Cases pass if all required claims are present, all citations are correct, the answer is on-topic, no fabricated content. Simple to aggregate; the pass rate is a meaningful metric.

### 5.2 The rubric dimensions

For clinical-domain cases, Meridian's rubric:

1. **Factually correct.** Every claim is supportable by the cited (or available) sources.
2. **Complete.** Addresses the actual question; required claims present.
3. **Cites correctly.** Citations point to real sources; cited content actually supports the claim.
4. **Appropriate.** Does not over-claim; does not hallucinate confidence; refuses when refusal is warranted.

Fail on any one = case fails.

The rubric is documented; the judge prompt enforces it; reviewers use it for calibration.

### 5.3 The rubric per case class

Different case classes may have different rubrics. A high-stakes clinical case has a stricter rubric than a low-stakes patient-education case:

- Clinical: factual correctness + citation accuracy required.
- Patient-education: factual correctness + reading-level appropriate.
- Refusal: refused-when-should + helpful-alternative-offered.

The case structure indicates which rubric applies (often by case-class tag).

### 5.4 The "must" vs "should" structure

The rubric distinguishes:

- **Must:** required for pass. Failure = case fails.
- **Should:** desired but not required. Tracked but does not fail the case.

Example: must include both required citations; should use the patient's preferred language (preferred but not failure if absent).

Tracking the should-dimensions provides quality signal that does not gate; the team can see "we're meeting all musts but missing 30% of shoulds — should we tighten the rubric?"

---

## 6. Ownership and refresh discipline

The golden set is a living artifact. The discipline keeps it relevant.

### 6.1 Per-case ownership

Each case has an owner — either the team that added it or the team responsible for the case class. The owner is accountable for:

- Reviewing the case when the system evolves (does this case still test what it was meant to test?).
- Updating expected outputs when the correct answer changes (clinical guidelines update; expected answer updates).
- Retiring the case if it becomes obsolete.

### 6.2 Per-suite ownership

A suite owner (team or person) is accountable for:

- Overall suite health.
- Coverage assessment (are the four selection criteria balanced?).
- Growth coordination (which cases to add this quarter?).
- Retirement coordination.

For Meridian: ai-platform-eng owns the overall eval-engineering practice; clinical-knowledge-engineering owns the clinical golden set; security-eng owns the adversarial subset.

### 6.3 The refresh cadence

Quarterly: every active case is reviewed by its owner:

- Is the case still representative of the workload?
- Is the expected answer still correct?
- Are the case-class tags still accurate?
- Should this case be retired, updated, or promoted to a more critical tier?

Cases that fail review are flagged. The team disposes: update, retire, re-assign.

### 6.4 The trigger-based refresh

In addition to quarterly:

- When the corpus changes meaningfully (e.g., a new edition of clinical guidelines lands): clinical cases are reviewed for expected-answer drift.
- When a new model version is adopted: cases with model-specific assumptions are reviewed.
- When a new feature ships: cases for that feature are added.
- When a production bug is fixed: a regression case is added (per [regression-eval-suites.md](./regression-eval-suites.md)).

### 6.5 The orphan detection

Quarterly scan: cases whose owners are no longer on the team. These are orphans. The team disposes: re-assign to a current team, or retire.

Without orphan detection, cases accumulate that nobody understands. The discipline detects.

### 6.6 The decay metric

Per-case freshness is tracked: when was this case last reviewed? Cases with > 9 months since last review are flagged.

The freshness metric is itself an SLI for suite health.

---

## 7. Growth patterns

The golden set grows from 20 to 500+ cases over time. The growth is structured.

### 7.1 Growth from dogfooding

During pre-production dogfooding, engineers use the system and surface cases:

- Cases where the system gave a wrong answer.
- Cases where the system surprised the engineer (good or bad).
- Cases that are representative of the engineer's intended use.

Each becomes a candidate golden-set case. The owner-team reviews; accepted cases are added with appropriate tags.

### 7.2 Growth from production feedback

Production interactions produce signal (per [online-eval-and-feedback.md](./online-eval-and-feedback.md)):

- Thumbs-down + free-text from users.
- Implicit signals (retry, edit, abandonment).
- Sampled online judge runs that flag cases.

The team reviews high-signal production cases weekly; promotes high-value ones to the golden set.

### 7.3 Growth from domain expertise

Domain experts (clinicians, lawyers, financial analysts depending on workload) contribute cases:

- Cases they would test if evaluating the system.
- Cases that come up in their domain practice.
- Cases that represent edge conditions experts know but engineers don't.

These cases are often the highest-value additions; the experts know the workload's structure in ways engineering teams do not.

### 7.4 Growth from production bugs

Every production bug becomes a regression case. Per [regression-eval-suites.md](./regression-eval-suites.md), the regression suite is a sibling structure to the golden set; some teams merge them, others keep them separate. Either pattern works.

### 7.5 The growth review

Quarterly: the suite owner reviews growth:

- How many cases added this quarter?
- From what sources?
- Coverage balance — are the four criteria still balanced?
- Which classes need more cases?

The growth is steered, not just accepted.

### 7.6 The de-duplication discipline

Growth produces near-duplicates. The discipline:

- Before adding a new case, search the suite for similar cases.
- If a similar case exists: update the existing case rather than adding a duplicate (unless the new case tests something genuinely different).
- Quarterly de-duplication pass: cluster cases; flag clusters; consolidate.

---

## 8. Integration with the broader eval practice

The golden set is consumed by multiple eval components.

### 8.1 The judge

Per [llm-as-judge-patterns.md](./llm-as-judge-patterns.md), the judge scores produced answers against expected_answer + scoring rubric from the case structure.

### 8.2 The eval gate

Per [eval-gate-architecture.md](./eval-gate-architecture.md), the gate runs the suite (or a fast subset) on every PR; blocks merge on regression.

### 8.3 The regression suite

Per [regression-eval-suites.md](./regression-eval-suites.md), the regression suite is either a tagged subset of the golden set or a separate suite.

### 8.4 The online judge

Per [online-eval-and-feedback.md](./online-eval-and-feedback.md), the online judge runs against production traffic; the golden-set rubric is the reference for scoring.

### 8.5 The dashboard

Per-class pass rates, trend lines, freshness metrics all derive from the golden set's structure.

The golden set's structure (case-class tags, scoring rubric, ownership) is what makes all of this possible. Without the structure, the consumers cannot do their jobs.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The clinical golden set

The Meridian Care Coordinator's clinical golden set has 200 cases at production maturity:

| Source | Count |
|---|---|
| Designed by clinical-knowledge-engineering during initial design | 60 |
| Added during dogfooding by clinical staff at pilot hospitals | 45 |
| Promoted from production feedback (thumbs-down + free-text review) | 38 |
| Regression cases from production incidents | 28 |
| Domain-expert contributions (consulting clinicians) | 29 |

The distribution across selection criteria:
- Representative: ~50 cases (25%).
- Edge: ~55 cases (27%).
- Known-failure: ~45 cases (22%).
- High-stakes: ~50 cases (25%).

The balance is healthy.

### 9.2 The case-class coverage

Per question_type:
- Lookup: 80 cases (40%).
- Multi-step / coordination: 50 cases (25%).
- Drug-interaction: 30 cases (15%, also overlaps with the separate drug-interaction subset).
- Multi-hop clinical reasoning: 25 cases (12.5%).
- Refusal / escalation: 15 cases (7.5%).

Per stakes:
- Critical (clinical-decision-support): 50 cases (25%).
- High (drafting / coordination): 60 cases (30%).
- Medium (lookup / scheduling): 70 cases (35%).
- Low (acknowledgment / formatting): 20 cases (10%).

The coverage matches production traffic distribution; cases are not over-weighted toward low-stakes lookup.

### 9.3 The case structure example

```yaml
case_id: CLIN-0042
question: "What is the post-discharge follow-up protocol for a CHF patient on the new Mercy Cleveland pathway?"

expected_sources:
  - chunk_id: "clinical-guideline:aha-hf-2024:section-3.2:chunk-0042"
    is_required: true
  - chunk_id: "tenant-protocol:mercy-cleveland:hf-22:chunk-0007"
    is_required: true

required_claims:
  - claim: "7-day nursing check-in is required"
    must_cite: "clinical-guideline:aha-hf-2024"
  - claim: "14-day clinician visit is required"
    must_cite: "clinical-guideline:aha-hf-2024"
  - claim: "Mercy Cleveland Protocol HF-22 applies"
    must_cite: "tenant-protocol:mercy-cleveland:hf-22"

expected_answer: |
  For a CHF patient on the new Mercy Cleveland pathway, post-discharge
  follow-up consists of:
  - 7-day nursing check-in (per AHA 2024 HF Discharge Bundle)
  - 14-day clinician visit (per AHA 2024 HF Discharge Bundle)
  Hospital-specific: Mercy Cleveland Protocol HF-22 includes specific
  attestation requirements for the 7-day check-in.

class_tags:
  question_type: lookup
  domain: clinical
  complexity: simple
  stakes: high
  production_frequency: common
  failure_history: never_failed

tenant_context:
  tenant_id: mercy-cleveland
  caller_role: rn

source:
  added_date: 2026-03-15
  added_by: clinical-knowledge-engineering
  added_via: representative_clinical_lookup
  last_reviewed: 2026-05-15
  reviewer: clinical-knowledge-engineering

scoring:
  rubric: meridian_clinical_rubric_v2
  must:
    factually_correct: true
    cites_correctly: true
    complete: true
    appropriate: true
  should:
    uses_preferred_language: true
    mentions_clinical_context: true
```

The structure is comprehensive but readable; each field has a purpose.

### 9.4 The refresh cycle

Quarterly: clinical-knowledge-engineering reviews the 200 clinical cases. Recent reviews:
- 2026-Q1: 8 cases updated (AHA 2024 guideline revisions); 3 cases retired (outdated drug references); 4 new cases added (post-incident regressions).
- 2026-Q2 (in progress): scheduled completion 2026-05-30.

Trigger-based refresh:
- When AHA 2024 guidelines released their Q1 update, ~12 cases referenced the affected sections; all reviewed within 2 weeks of the update.
- When the Mercy Cleveland Protocol HF-22 was revised, 1 case was updated.

### 9.5 The orphan detection

Quarterly scan in 2026-Q2 flagged:
- 3 cases owned by an engineer who left the team. Re-assigned to clinical-knowledge-engineering.
- 2 cases not reviewed in 12 months. Force-reviewed; both still valid.

The detection ran in ~10 minutes against a structured query; the dispositions were one-PR each.

### 9.6 The platform discipline

- Golden set cases in version control (`meridian-eval/cases/`).
- PR-reviewed additions; owner-team approval required.
- Quarterly refresh review.
- Quarterly orphan-detection scan.
- Per-class coverage dashboard.

---

## 10. Anti-patterns

### 10.1 "Cases accumulated organically"

The golden set has 80 cases, added over years by various engineers. Nobody knows what the suite covers. New cases are added without checking for duplication; the suite is unmaintainable.

**Corrective.** Inventory cases against the four selection criteria; tag for class coverage; retire duplicates and orphans; establish ownership.

### 10.2 "No case-class taxonomy"

Cases have IDs and questions but no class tags. Per-class analysis is impossible.

**Corrective.** Taxonomy per section 3; tag every case.

### 10.3 "Expected answers are strings to match exactly"

The scoring is exact-string-match against expected_answer. Models produce variations; the eval fails on equivalent answers; teams either lower the bar or live with high false-positive failure rates.

**Corrective.** Expected_answer is a semantic reference; the judge decides equivalence; scoring rubric specifies must / should structure.

### 10.4 "No ownership"

Cases exist; nobody is accountable for them. Cases decay; updates do not happen; orphan cases accumulate.

**Corrective.** Per-case ownership; per-suite ownership; quarterly review.

### 10.5 "Growth without curation"

Cases get added; nobody coordinates; the suite grows past 1,000 cases with poor class balance and many near-duplicates.

**Corrective.** Quarterly growth review per section 7.5; de-duplication pass.

### 10.6 "High-stakes cases under-represented"

The suite is heavily lookup / representative cases (easy to write); high-stakes cases (harder to design correctly) are absent. The eval over-predicts production quality on high-stakes scenarios.

**Corrective.** Balance per section 2.1; high-stakes cases get explicit allocation.

### 10.7 "Refresh deferred indefinitely"

The team built the suite once; never refreshes it. Cases become obsolete (corpus changed, expected answer no longer correct); pass rate is meaningless because the rubric is stale.

**Corrective.** Quarterly refresh cadence; trigger-based refresh on corpus / model changes.

### 10.8 "Cases for which the team cannot articulate the purpose"

A new engineer asks "why is this case in the suite?" Nobody can answer. The case persists because nobody dares retire it.

**Corrective.** Source metadata on every case (why was it added); quarterly review prunes cases without clear purpose.

---

## 11. Findings (sprint-assignable)

### GOLDEN-001 — Severity: Critical
**Finding.** No golden set exists; eval is ad-hoc.
**Recommendation.** Build the 20-case starting set per section 2.3; balanced across the four selection criteria.
**Owner.** ai-platform-eng + domain experts, sprint N+1.

### GOLDEN-002 — Severity: Critical
**Finding.** Golden set has no case-class taxonomy; per-class analysis is impossible.
**Recommendation.** Taxonomy per section 3; tag every case.
**Owner.** ai-platform-eng, sprint N+1.

### GOLDEN-003 — Severity: High
**Finding.** Cases have no owner; the suite decays without accountability.
**Recommendation.** Per-case ownership; per-suite ownership.
**Owner.** ai-platform-eng team lead, sprint N+1.

### GOLDEN-004 — Severity: High
**Finding.** Expected_answer is treated as exact-match target; false-positive failure rate is high.
**Recommendation.** Semantic reference per section 4.3; judge decides equivalence; scoring rubric specifies criteria.
**Owner.** ai-platform-eng, sprint N+2.

### GOLDEN-005 — Severity: High
**Finding.** The four selection criteria are not balanced; high-stakes cases under-represented.
**Recommendation.** Audit per section 2.1; add cases to balance.
**Owner.** ai-platform-eng + domain experts, sprint N+2.

### GOLDEN-006 — Severity: High
**Finding.** Quarterly refresh is not scheduled; cases decay without review.
**Recommendation.** Refresh cadence per section 6.3; trigger-based refresh on corpus / model changes.
**Owner.** ai-platform-eng team lead, sprint N+2.

### GOLDEN-007 — Severity: High
**Finding.** No source metadata on cases; the team cannot articulate why each case is in the suite.
**Recommendation.** Source field per case per section 4.1; quarterly review prunes purposeless cases.
**Owner.** ai-platform-eng, sprint N+2.

### GOLDEN-008 — Severity: Medium
**Finding.** Orphan cases accumulate; cases owned by ex-team-members persist.
**Recommendation.** Quarterly orphan scan per section 6.5; re-assign or retire.
**Owner.** ai-platform-eng team lead, sprint N+3.

### GOLDEN-009 — Severity: Medium
**Finding.** Growth is unmanaged; suite contains near-duplicates.
**Recommendation.** Growth review per section 7.5; quarterly de-duplication pass.
**Owner.** ai-platform-eng, sprint N+3.

### GOLDEN-010 — Severity: Medium
**Finding.** Domain experts do not contribute cases; the eval suite reflects engineering's view rather than the workload's.
**Recommendation.** Domain-expert contribution channel; quarterly review with domain experts.
**Owner.** ai-platform-eng + domain experts, sprint N+3.

### GOLDEN-011 — Severity: Medium
**Finding.** Case-class coverage is not visualized; team cannot see where the suite is weak.
**Recommendation.** Per-class coverage dashboard per section 9.2.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### GOLDEN-012 — Severity: Medium
**Finding.** The suite has grown past the practical ceiling; runs are too slow / expensive.
**Recommendation.** Consolidate per section 2.5; retire low-value cases.
**Owner.** ai-platform-eng, sprint N+3.

### GOLDEN-013 — Severity: Medium
**Finding.** Case freshness is not tracked; old cases persist without review.
**Recommendation.** Per-case last_reviewed field; freshness dashboard; alert on > 9 months stale.
**Owner.** ai-platform-eng, sprint N+4.

### GOLDEN-014 — Severity: Medium
**Finding.** Rubric is informal; reviewers apply different standards.
**Recommendation.** Documented rubric per section 5.2; judge prompt enforces; calibration discipline.
**Owner.** ai-platform-eng, sprint N+4.

### GOLDEN-015 — Severity: Medium
**Finding.** Should-dimensions are not tracked; soft quality signals are invisible.
**Recommendation.** Must / should structure per section 5.4; aggregate both.
**Owner.** ai-platform-eng, sprint N+4.

### GOLDEN-016 — Severity: Low
**Finding.** Case ID convention is inconsistent; cases use multiple ID formats.
**Recommendation.** Standardize per section 4.5; migrate non-conforming.
**Owner.** ai-platform-eng, sprint N+5.

### GOLDEN-017 — Severity: Low
**Finding.** Conversational cases are flat single-turn; multi-turn coverage is absent.
**Recommendation.** Multi-turn case structure per section 4.2.
**Owner.** ai-platform-eng, sprint N+5.

### GOLDEN-018 — Severity: Low
**Finding.** Golden set documentation is thin; new contributors cannot understand the suite's structure.
**Recommendation.** Generated documentation from case schema; commit alongside suite.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team starting from "we don't have a golden set":

- [ ] **Sprint 0 — design.** Identify the workload's case classes. Define the taxonomy per section 3.
- [ ] **Sprint 1 — first 20 cases.** Build the minimum viable golden set per section 2.3, balanced across the four selection criteria.
- [ ] **Sprint 1 — case structure.** Define the case structure per section 4. Commit cases as version-controlled artifacts.
- [ ] **Sprint 1 — ownership.** Assign owner per case and per suite. Document.
- [ ] **Sprint 2 — scoring rubric.** Document the rubric per section 5. Binary pass/fail default.
- [ ] **Sprint 2 — first eval run.** Manual review per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 3.4; capture failure modes.
- [ ] **Sprint 3 — growth process.** Establish the growth channels per section 7; coordinate growth.
- [ ] **Sprint 3 — coverage dashboard.** Per-class coverage visualization.
- [ ] **Sprint 4 — refresh cadence.** Quarterly review on the calendar; trigger-based refresh defined.
- [ ] **Sprint 4 — orphan detection.** Quarterly orphan scan.
- [ ] **Ongoing — discipline.** Growth review; freshness tracking; coverage assessment.

A team that completes this sequence has a golden set that is deliberate, balanced, owned, and refreshed. A team without this discipline carries an accumulated grab-bag that nobody trusts.

---

## 13. References

- This repo: [eval-engineering/eval-engineering-playbook.md](./eval-engineering-playbook.md) — the broader practice this is the depth on.
- This repo: [eval-engineering/llm-as-judge-patterns.md](./llm-as-judge-patterns.md) — the judge that consumes the case rubric.
- This repo: [eval-engineering/regression-eval-suites.md](./regression-eval-suites.md) — the sibling structure for production-bug-driven cases.
- This repo: [eval-engineering/online-eval-and-feedback.md](./online-eval-and-feedback.md) — growth channel from production feedback.
- This repo: [eval-engineering/eval-gate-architecture.md](./eval-gate-architecture.md) — the CI integration.
- This repo: [eval-engineering/eval-of-rag.md](./eval-of-rag.md) — RAG-specific case structure extensions.
- This repo: [eval-engineering/eval-of-agents.md](./) (coming) — agent-specific case structure extensions.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked architecture this eval suite supports.
- Hugging Face evaluate library, OpenAI Evals, LangSmith Evals, Braintrust — case format and tooling references.
