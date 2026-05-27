# Labeling and Annotation

> **Audience.** Engineers building labeling pipelines for AI workloads. Tech leads whose vendor-labeled data quality is mysteriously dropping. Anyone who has heard "we'll just hire a labeling vendor" and is realizing that's not the discipline. **Scope.** The *engineering* practice of labeling workflow design: rubric, guidelines, inter-annotator agreement (IAA), calibration, vendor-vs-internal-vs-hybrid, quality-control gates, failure modes (rubric drift, guideline ambiguity, annotator fatigue, gaming). Not the data quality discipline overall (see [data-quality-for-ai.md](./data-quality-for-ai.md), companion). Not the synthetic data generation alternative (see [synthetic-data-generation.md](./synthetic-data-generation.md) *(coming)*). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Labeling is engineering. The teams that treat it as a procurement problem (hire a vendor; receive labels) discover that labeled data quality determines model quality, and quality decays without engineering discipline.

The pattern:

- Vendor labeled 10,000 documents.
- Initially: 85% accuracy on internal review.
- 3 months later: 70% accuracy.
- Models trained on this data degrade.
- Cause: rubric drift; annotator fatigue; new annotators without calibration.

Engineering the labeling workflow:

- Rubric (clear definition of correct labels).
- Guidelines (how to apply the rubric to edge cases).
- Inter-annotator agreement metric (when annotators disagree, the rubric is unclear).
- Calibration (annotators trained against gold-standard cases).
- Quality gates (sampling + review).
- Feedback loop (rubric updates based on what's confusing).

This document covers each.

This document is opinionated about four things:

1. **Labeling is an engineering workflow, not a procurement contract.** Treat it that way.
2. **Inter-annotator agreement is the load-bearing metric.** Without it, you don't know if the rubric is unclear or annotators are diverging.
3. **Vendor labels need internal quality control.** Trust but verify.
4. **The rubric is a versioned artifact.** Cross-link to [dataset-versioning.md](./dataset-versioning.md).

Structure: (2) the rubric design; (3) guidelines and edge cases; (4) inter-annotator agreement (IAA); (5) calibration; (6) vendor vs internal vs hybrid; (7) quality control gates; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The rubric design

What the rubric is and how to design it.

### 2.1 The rubric

A rubric defines what a "correct" label is:

```
For document classification:
  Label "clinical_referral" if the document is:
    - A referral letter from a clinician.
    - Recommending a patient see a specialist.
    - In any clinical specialty.
  
  Label "discharge_summary" if the document is:
    - A summary of a hospital stay or ER visit.
    - Authored by clinicians involved in care.
  ...
```

Explicit; rule-based.

### 2.2 The rubric levels

For different workloads:

- Binary (yes/no).
- Categorical (one of N classes).
- Ordinal (rating 1-5).
- Free-form (annotation text).

Each has different rubric considerations.

### 2.3 The "what's the unit" question

Per task, what's the unit being labeled:

- Per-document (classification).
- Per-sentence (sentiment).
- Per-entity (NER).
- Per-image (computer vision).

Defines the rubric's scope.

### 2.4 The rubric explicitness

Rubric should be:

- Specific (not "is this important?" but "does this require clinician action within 24 hours?").
- Exhaustive (covers most cases).
- Edge-case aware (covers ambiguous cases).

Vague rubric = inconsistent labels.

### 2.5 The rubric simplicity

Too complex:

- Annotators can't remember.
- Disagreement on edge cases.

Too simple:

- Misses important nuance.

Aim: 5-15 categories or rules; clear; testable.

### 2.6 The rubric examples

Each rubric category has examples:

```
clinical_referral examples:
  "Dr. Smith refers patient John Doe (DOB 1955-01-15) to Dr. Jones for cardiology evaluation."
  
discharge_summary examples:
  "Hospital course: Patient admitted with chest pain..."
```

Examples concrete; annotators reference.

### 2.7 The "what is the rubric NOT" exclusion

For each rubric category:

- What does NOT fall under this.
- Boundary clarification.

Helps annotators draw lines.

### 2.8 The rubric as versioned artifact

The rubric changes over time:

- v1.0.0: initial.
- v1.1.0: refined "clinical_referral" definition.
- v2.0.0: added new category.

Cross-link to [dataset-versioning.md §2.4](./dataset-versioning.md).

### 2.9 The rubric review

Per quarter:

- Are annotators confused? Where?
- Does the rubric match production needs?
- Refinements?

Iterative.

---

## 3. Guidelines and edge cases

How to apply the rubric to ambiguous cases.

### 3.1 The guidelines document

For each rubric category, additional guidance:

```markdown
# Clinical Referral Labeling Guidelines

## Boundary cases
- Referrals to imaging (radiology, MRI) → still clinical_referral.
- Referrals to lab work → NOT clinical_referral (use lab_order category).
- Self-referrals → NOT clinical_referral.

## Edge cases
- Letter that mentions referral but is primarily summary → discharge_summary.
- Multi-purpose letters → use primary purpose.

## When uncertain
- Mark as needs_review.
```

Resolves ambiguity.

### 3.2 The "what to do when uncertain" instruction

Always: a needs_review label.

Better to have a clear "needs review" than fabricated certainty.

### 3.3 The annotator-Q&A

A channel where annotators ask questions:

- Slack channel; or email list.
- Lead annotator (or product / SME) responds.
- Q&As preserved; inform guideline updates.

Captures rubric ambiguities.

### 3.4 The guideline-update flow

When ambiguity is found:

- Discuss with annotators.
- Update the guidelines.
- Re-train all annotators.
- Re-label affected cases (if material).

Iterative.

### 3.5 The "we have 100 edge cases now" maturity

After 6-12 months of labeling:

- Many edge cases documented.
- Guidelines mature.

This is normal; it's the discipline working.

### 3.6 The "we changed the guidelines mid-project" complication

If guidelines change after labeling has started:

- Previously labeled data needs re-review.
- Or accept inconsistency.

For substantive changes: re-label.

### 3.7 The "what to do with disagreements" resolution

When annotators disagree:

- Adjudicator (senior annotator) decides.
- Or majority vote.
- Or flag for SME review.

Per-rubric policy.

### 3.8 The "the rubric requires SME judgment" reality

For clinical / specialized labeling:

- Annotators may lack expertise.
- SME (subject matter expert) reviews.

Either: SMEs do all labeling (expensive); or SMEs review subset / edge cases.

---

## 4. Inter-annotator agreement (IAA)

The metric that says "is the rubric clear."

### 4.1 The IAA metric

For each task, measure agreement:

- **Cohen's kappa** (2 annotators).
- **Fleiss' kappa** (more annotators).
- **Percent agreement** (simple).

Higher = better.

```
Cohen's kappa:
  0.81-1.00: almost perfect agreement
  0.61-0.80: substantial agreement
  0.41-0.60: moderate agreement
  0.21-0.40: fair agreement
  0.01-0.20: slight agreement
```

Target: > 0.7 (substantial agreement).

### 4.2 The IAA measurement

Have multiple annotators label the same items:

- 5-10% of cases double-labeled.
- Compute IAA on those.

Statistical.

### 4.3 The "IAA is low" diagnostic

Low IAA means:

- Rubric is ambiguous.
- Guidelines insufficient.
- Annotators not calibrated.

Investigate per cause.

### 4.4 The IAA per category

For categorical rubrics:

- Per-category IAA.
- Some categories may have lower agreement.

Focus refinement on low-agreement categories.

### 4.5 The IAA trend

Over time:

- IAA improving: training / rubric maturing.
- IAA declining: drift or annotator fatigue.

Track.

### 4.6 The "we have IAA but not stable annotators" issue

If annotators turn over, IAA can drop:

- New annotators less calibrated.
- Re-calibration needed.

Track per-annotator metrics.

### 4.7 The "we set the threshold but never met it" outcome

If target IAA never achieved:

- Rubric may be too ambiguous; refine.
- Or task is inherently subjective; accept lower.

Some tasks have lower-bound IAA (e.g., creative writing quality: hard to agree).

### 4.8 The "the SME-vs-annotator gap" pattern

SMEs may not agree with annotators:

- SME knows nuances annotators miss.
- IAA between SME and annotators is the real test.

Calibrate annotators to SMEs.

### 4.9 The IAA in production

For production labeling pipelines:

- Sample IAA per batch.
- Alert if drops.

Continuous metric.

---

## 5. Calibration

How annotators get aligned.

### 5.1 The calibration process

For new annotators:

1. Read rubric + guidelines.
2. Label N gold-standard cases.
3. Compare to gold-standard answers.
4. Discuss disagreements.
5. Re-label.
6. Iterate until aligned.

Like a training certification.

### 5.2 The gold-standard set

A curated set of cases:

- Each with the "correct" label.
- Validated by SMEs.
- Used for annotator calibration.

50-200 cases typical.

### 5.3 The calibration gates

Annotators must pass calibration before producing data:

- Achieve agreement with gold-standard on 90%+ of cases.
- Discussion of disagreement areas.
- Approved.

Quality gate.

### 5.4 The ongoing calibration

Periodically (monthly or quarterly):

- Re-test annotators against gold-standard.
- Catch drift.
- Re-calibrate if needed.

Maintenance.

### 5.5 The "we onboarded a new annotator without calibration" risk

Without calibration:

- Their labels are not consistent with the team.
- Data quality suffers.

Don't ship un-calibrated annotators.

### 5.6 The "vendor annotators rotate frequently" challenge

For vendors:

- Annotators change.
- Each new annotator needs calibration.

Mitigation:

- Vendor onboarding includes calibration.
- Vendor agrees to calibration before labeling.

Contract terms.

### 5.7 The cross-vendor calibration

If using multiple vendors:

- Each calibrates against same gold-standard.
- IAA between vendors should be > threshold.

Cross-vendor consistency.

### 5.8 The "calibration takes longer than expected" warning

Calibration is ~1-2 days per annotator:

- Per annotator costs.
- Schedule accordingly.

### 5.9 The calibration evolution

Over time:

- Calibration set grows (more edge cases).
- Calibration process refined.

Iterative.

---

## 6. Vendor vs internal vs hybrid

How to staff labeling.

### 6.1 The internal-team option

Hire / re-purpose internal staff:

**Pros.**
- Domain expertise (especially clinical / legal / financial).
- Direct accountability.
- Stable team.

**Cons.**
- Slow to scale.
- Expensive per labeled item.
- Limited capacity.

**When right.** High-domain-expertise workloads; smaller volume.

### 6.2 The vendor option

Engage labeling vendor (Scale AI, Surge, Labelbox, etc.):

**Pros.**
- Scale (thousands of annotators).
- Fast turnaround.

**Cons.**
- Distant annotators (limited domain insight).
- Quality variable.
- Annotator rotation.

**When right.** High-volume, lower-expertise workloads.

### 6.3 The hybrid option

Vendor for volume; internal for quality:

- Vendor does bulk labeling.
- Internal team reviews sample (e.g., 5-10%).
- Internal team handles edge cases.

**Pros.**
- Volume + quality.

**Cons.**
- Coordination overhead.

**When right.** Most production workloads with sufficient volume.

### 6.4 The decision criteria

```
Workload                Volume    Expertise needed   Recommendation
─────────────────────────────────────────────────────────────────────
Document classification High      Low-medium         Vendor or hybrid
Clinical coding         Medium    High               Internal or hybrid
Sentiment              High      Low                Vendor
Image annotation       Medium     Specialized        Vendor
SME review of vendor    Selective  High               Internal
```

Per workload.

### 6.5 The vendor evaluation

Before engaging:

- Calibrate against gold-standard.
- IAA with internal team.
- Cost per label.
- Turnaround time.

Choose based on data.

### 6.6 The vendor management

Ongoing:

- IAA monitoring.
- Periodic audits.
- Communication channel.

Active management.

### 6.7 The "vendor quality dropped" response

If quality decline detected:

- Diagnose (rubric change? annotator rotation?).
- Vendor discussion.
- Re-calibration.
- Escalation if persistent.

Cross-link to §4.5 and §5.6.

### 6.8 The cost trade-off

Internal:
- $50-100/hour per annotator.
- ~$10-30 per labeled item.

Vendor:
- $0.50-5 per labeled item (volume-dependent).

10-50x cost difference; quality / coordination overhead may close some gap.

### 6.9 The "we should use SMEs as annotators" trap

SMEs are expensive; their time is high-value:

- Use SMEs for: gold-standard creation; edge cases; vendor calibration.
- Use vendors / internal annotators for: bulk labeling.

SMEs scale poorly.

---

## 7. Quality control gates

The gates between labeling and production use.

### 7.1 The pre-production review

Before labeled data is used:

- Sample (5-10% of cases).
- Internal review against gold-standard.
- Pass rate must exceed threshold.

Quality gate.

### 7.2 The release gate

Before a labeled dataset is "released":

- IAA above threshold.
- Sample review pass.
- Changelog complete.

Then: dataset versioned and released.

### 7.3 The post-deployment monitoring

Once labels are used (e.g., for fine-tuning):

- Model performance vs eval.
- If model regresses: investigate labels.

Indirect quality check.

### 7.4 The "we labeled 10k; QC sampled 500" coverage

Coverage trade-off:

- Higher coverage (review all 10k): expensive; quality verified.
- Lower (sample 500): efficient; statistical inference.

For high-stakes: higher coverage.
For routine: sample.

### 7.5 The error budget

Acceptable error rate per workload:

- Clinical: <1% error.
- Customer support: 5-10% error.
- Marketing: 15-20% error.

Per workload.

### 7.6 The "we found errors; what now" recovery

If QC finds errors:

- Quantify (sample more).
- Diagnose (cause).
- Re-label affected cases.
- Re-calibrate annotators if needed.
- Update guidelines if cause is rubric ambiguity.

Iterative.

### 7.7 The continuous-QC pattern

Don't wait for final QC:

- Sample 10-20% of cases continuously.
- Daily / weekly review.
- Issues caught early.

Continuous quality.

### 7.8 The vendor-side QC

Vendors should do QC too:

- Internal vendor review.
- Vendor's IAA metrics.

Get vendor's QC data alongside their labels.

---

## 8. Worked Meridian example

Meridian's labeling discipline.

### 8.1 The labeling workloads

- Document classification (40k docs/month).
- Clinical note extraction (5k/month).
- Care plan annotation (1k/month).
- Eval golden set creation (~100 cases/month).

Mixed: vendor for high-volume, internal SME for high-expertise.

### 8.2 The document classification setup

- Volume: high.
- Expertise: clinical knowledge helpful but not deep.
- Approach: hybrid.
  - Vendor (Surge AI) for bulk labeling.
  - Internal team for QC + edge cases.

Workflow:

1. Documents to vendor.
2. Vendor labels (5,000/week).
3. 10% sample to internal review.
4. Internal flags errors; vendor re-labels.
5. Vendor IAA tracked.

### 8.3 The rubric for document classification

15 categories:

- clinical_referral.
- discharge_summary.
- progress_note.
- consultation_report.
- lab_result.
- imaging_report.
- medication_list.
- ... etc.

Each with examples + boundary cases.

Versioned: rubric-v5.4.0 in production.

### 8.4 The clinical extraction (internal SMEs)

- Volume: lower.
- Expertise: deep clinical needed.
- Approach: internal SMEs (CMOs / Cl-Inf team).

Rubric: per-extraction-type (medications, allergies, vitals).

### 8.5 The IAA tracking

Document classification:

- Cohen's kappa: 0.78 (substantial).
- Target: > 0.7.

Clinical extraction:

- Per extraction type; varies (0.85-0.92).

Tracked monthly.

### 8.6 The Q1 2026 quality decline

Vendor-labeled document classification quality declined from 92% to 85% over 6 weeks.

Investigation:

- New vendor annotators (rotation).
- Calibration not re-run.
- Rubric updates not communicated.

Fix:

- Re-calibrate all vendor annotators.
- Re-communicate rubric updates.
- Monthly vendor sync established.

Quality recovered.

### 8.7 The Q2 2026 rubric refinement

A rubric refinement was needed:

- "Progress note" boundary unclear.
- Internal review found 15% disagreement.

Process:

1. Document the ambiguity.
2. SME discussion.
3. Update guidelines.
4. Re-train annotators.
5. Re-label affected cases (3,000 documents).
6. Update rubric to v5.5.0.

Took 2 weeks; quality restored to 95%.

### 8.8 The vendor management

Weekly:

- Volume.
- IAA metric.
- Sample QC.

Monthly:

- Vendor IAA report.
- Cost reconciliation.
- Rubric / guideline updates.

Quarterly:

- Vendor contract review.
- Cost vs quality.

### 8.9 The infrastructure

- Labeling platform (Surge AI integration).
- IAA dashboard.
- QC sample tool.
- Rubric / guideline docs (Git).

Total ~$3k/month tooling + vendor cost.

### 8.10 The annotator cost

- Vendor: $0.80 per document (volume rate).
- Internal SME review: ~$5 per document (allocated time).
- Net: ~$1 per labeled document.

For 40k/month: $40k/month total.

### 8.11 The cost-vs-quality outcome

Hybrid approach:
- 5x more efficient than internal-only.
- Quality maintained at 95%.
- Suitable for the workload.

### 8.12 The lessons

- Vendor labeling needs active management.
- IAA is the load-bearing metric.
- Calibration must be ongoing (rotation).
- Rubric is a versioned artifact; refinement is normal.

---

## 9. Anti-patterns

### 9.1 The "hire a vendor and trust them" expectation

**Pattern.** Engage vendor; receive labels; assume quality.

**Corrective.** Active management per §6.6.

### 9.2 The IAA-not-measured gap

**Pattern.** Labels collected; agreement unknown.

**Corrective.** Per §4.

### 9.3 The unversioned rubric

**Pattern.** Rubric is in some doc; changes; no versioning.

**Corrective.** Versioned per §2.8.

### 9.4 The annotators-uncalibrated launch

**Pattern.** Annotators start labeling without gold-standard calibration.

**Corrective.** Calibration gate per §5.3.

### 9.5 The "we have IAA but ignore drift" complacency

**Pattern.** IAA was good at launch; not monitored over time.

**Corrective.** Continuous monitoring per §4.9.

### 9.6 The vendor-only-no-internal-QC blind trust

**Pattern.** Vendor's quality reports trusted; no internal verification.

**Corrective.** Internal QC per §7.

### 9.7 The "we don't have time to calibrate new annotators" rush

**Pattern.** New annotators start producing; quality drops.

**Corrective.** Calibration is non-optional per §5.5.

### 9.8 The SMEs-do-all-labeling overuse

**Pattern.** SMEs label everything; capacity bottleneck.

**Corrective.** Hybrid per §6.9.

### 9.9 The rubric-change-without-relabel

**Pattern.** Rubric updated; previously-labeled data not re-reviewed.

**Corrective.** Re-label material changes per §3.6.

### 9.10 The "labels are good; we don't need to track per-annotator" simplification

**Pattern.** Aggregate metrics OK; per-annotator drift invisible.

**Corrective.** Per-annotator metrics per §4.6.

---

## 10. Findings (sprint-assignable)

### DATA-LA-001 — Severity: Critical
**Finding.** Rubric not documented / versioned.
**Recommendation.** Per §2 and §2.8.
**Owner.** AI platform + data engineering, sprint N+1.

### DATA-LA-002 — Severity: Critical
**Finding.** IAA not measured.
**Recommendation.** Per §4.
**Owner.** AI platform + data engineering, sprint N+1.

### DATA-LA-003 — Severity: Critical
**Finding.** Annotators not calibrated.
**Recommendation.** Calibration gate per §5.3.
**Owner.** AI platform, sprint N+1.

### DATA-LA-004 — Severity: High
**Finding.** Vendor management informal.
**Recommendation.** Active management per §6.6.
**Owner.** AI platform + procurement, sprint N+2.

### DATA-LA-005 — Severity: High
**Finding.** Quality control gates absent.
**Recommendation.** Per §7.
**Owner.** AI platform, sprint N+2.

### DATA-LA-006 — Severity: High
**Finding.** Guidelines for edge cases absent.
**Recommendation.** Per §3.
**Owner.** AI platform + SMEs, sprint N+2.

### DATA-LA-007 — Severity: High
**Finding.** Re-calibration not scheduled (drift).
**Recommendation.** Per §5.4.
**Owner.** AI platform, sprint N+2.

### DATA-LA-008 — Severity: High
**Finding.** Per-annotator metrics absent.
**Recommendation.** Per §4.6.
**Owner.** data engineering, sprint N+2.

### DATA-LA-009 — Severity: Medium
**Finding.** Annotator-Q&A channel absent.
**Recommendation.** Per §3.3.
**Owner.** AI platform, sprint N+3.

### DATA-LA-010 — Severity: Medium
**Finding.** Annotator turnover not handled.
**Recommendation.** Vendor contract requires calibration per §5.6.
**Owner.** procurement + AI platform, sprint N+3.

### DATA-LA-011 — Severity: Medium
**Finding.** Rubric not refined based on disagreements.
**Recommendation.** Iterative refinement per §2.9.
**Owner.** AI platform + SMEs, sprint N+3.

### DATA-LA-012 — Severity: Medium
**Finding.** Cross-vendor IAA not measured.
**Recommendation.** Per §5.7.
**Owner.** AI platform, sprint N+3.

### DATA-LA-013 — Severity: Medium
**Finding.** Continuous QC absent (only final QC).
**Recommendation.** Per §7.7.
**Owner.** AI platform, sprint N+3.

### DATA-LA-014 — Severity: Medium
**Finding.** Re-labeling on rubric change not done.
**Recommendation.** Per §3.6.
**Owner.** AI platform, sprint N+4.

### DATA-LA-015 — Severity: Low
**Finding.** Annotator fatigue not monitored.
**Recommendation.** Per-annotator metric trend.
**Owner.** data engineering, sprint N+4.

### DATA-LA-016 — Severity: Low
**Finding.** Vendor quality reports not validated independently.
**Recommendation.** Per §7.8.
**Owner.** AI platform + procurement, sprint N+5.

### DATA-LA-017 — Severity: Low
**Finding.** Annotator gamification / fraud not checked.
**Recommendation.** Behavioral analysis (label patterns; speed).
**Owner.** data engineering, sprint N+5.

### DATA-LA-018 — Severity: Low
**Finding.** Cost-per-label not tracked over time.
**Recommendation.** Track; identify drift.
**Owner.** FinOps + procurement, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Document rubric per §2.**
- [ ] **Version rubric per §2.8.**
- [ ] **Write guidelines for edge cases per §3.**
- [ ] **Build gold-standard set per §5.2.**
- [ ] **Implement calibration gate per §5.3.**
- [ ] **Measure IAA per §4.**
- [ ] **Per-annotator metrics per §4.6.**
- [ ] **Continuous QC per §7.7.**
- [ ] **Vendor management cadence per §6.6.**
- [ ] **Annual rubric review per §2.9.**

---

## 12. References

**In this folder.**
- [dataset-versioning.md](./dataset-versioning.md) — rubric versioning.
- [data-quality-for-ai.md](./data-quality-for-ai.md) — quality metrics.
- [synthetic-data-generation.md](./synthetic-data-generation.md) *(coming)* — alternative to labeling.
- [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md) *(coming)* — labeling for eval.

**Elsewhere in this repo.**
- [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — gold-standard for calibration.
- [rag-engineering/retrieval-engineering.md](../rag-engineering/retrieval-engineering.md) — labeled data for relevance.

**Sibling repos.**
- [ai-architecture-reference-architecture / data-architecture-for-ai / lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md) — labeled-data provenance.

**External.**
- Cohen's kappa / Fleiss' kappa literature.
- Labeling platform documentation (Scale AI, Surge, Labelbox).
- Annotation best-practices literature.
