# Data Contracts for AI

> **Audience.** Engineers whose AI pipeline broke because an upstream data source changed schema without notice. Tech leads whose retrieval corpus has gotten stale and they don't know why. Anyone whose "data engineering team" is "another team that does data stuff" with unclear interface. **Scope.** The *engineering* practice of data contracts: treating upstream data sources as contract-bound suppliers; schema; freshness; content-type guarantees; change-notification protocol; contract-violation alerting; integration with the architecture sibling's data-architecture. Not the broader data engineering (sources vs corpus is in [retrieval-corpus-engineering.md](./retrieval-corpus-engineering.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Data dependencies are integration boundaries. Without contracts:

- Upstream changes schema without warning.
- AI pipeline breaks.
- Data flows in unexpected formats.
- "What does this field actually mean?" becomes unclear.

With data contracts:

- Schema explicit.
- Freshness SLAs.
- Change-notification protocol.
- Quality guarantees.

Same discipline that mature service APIs have, applied to data sources.

This document covers the engineering practice.

This document is opinionated about four things:

1. **Data sources have contracts.** Schema, freshness, type. Documented.
2. **Contract violations trigger alerts.** Pre-empt downstream failures.
3. **Change notification is the supplier's obligation.** Don't discover schema changes via broken pipelines.
4. **Contracts integrate with the broader data-architecture.** Cross-link to architecture sibling.

Structure: (2) the contract elements; (3) schema contracts; (4) freshness contracts; (5) content-type contracts; (6) change-notification protocol; (7) contract violation handling; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The contract elements

What's in a contract.

### 2.1 Schema

What fields, what types:

```yaml
contract: ehr-clinical-notes
schema:
  patient_id: string (required)
  encounter_id: string (required)
  note_text: string (required)
  date_authored: ISO 8601 datetime (required)
  author_npi: string (required)
  note_type: enum [progress|consultation|admission|discharge|other] (required)
  metadata: object (optional)
```

Per data source.

### 2.2 Freshness

How fresh:

```yaml
freshness:
  max_staleness: < 24 hours (P99)
  update_frequency: nightly batch + real-time additions
  delivery_window: 02:00-04:00 UTC daily
```

SLA.

### 2.3 Content-type guarantees

Quality dimensions:

- Encoding (UTF-8).
- No PII outside expected fields.
- No HTML in plain-text fields.
- Etc.

Per field.

### 2.4 Volume guarantees

Expected volume:

- 800 records/day average.
- 200-1500 record range.
- Alert if outside.

Capacity planning.

### 2.5 The contract structure

Per contract:

```yaml
contract:
  source: EHR clinical notes
  supplier: EHR team
  consumer: AI platform (Care Coordinator, Patient API)
  schema: [...]
  freshness_sla: [...]
  content_guarantees: [...]
  volume_expectations: [...]
  change_notification_lead_time: 30 days
  violation_response: documented runbook
  contract_owner: AI platform
  supplier_owner: EHR team
  contract_version: v3.2
  effective_from: 2026-01-01
```

Comprehensive.

### 2.6 The "contract is documentation" minimum

At minimum:

- Schema documented.
- Freshness expected.
- Change-notification expected.

Better than nothing.

### 2.7 The "contract is enforced" maturity

Mature platform:

- Schema validated at ingestion.
- Freshness measured.
- Violations alerting.
- Negotiation channel.

Active management.

### 2.8 The per-source contracts

Each source: its own contract.

- EHR notes.
- Formulary.
- Guidelines.
- Customer data.
- Third-party APIs.

Per source.

### 2.9 The contract repository

Per organization:

- All contracts cataloged.
- Versioned.
- Discoverable.

Cross-link to [dataset-versioning.md](./dataset-versioning.md).

---

## 3. Schema contracts

The structural agreement.

### 3.1 Schema as code

Schemas formal:

```python
@dataclass
class ClinicalNoteContract:
    patient_id: str  # required
    encounter_id: str  # required
    note_text: str  # required
    date_authored: datetime  # required
    author_npi: str  # required (NPI format)
    note_type: Literal["progress", "consultation", "admission", "discharge", "other"]  # required
    metadata: Optional[dict]  # optional
```

Type-safe; validatable.

### 3.2 Schema validation

At ingestion:

```python
def validate_clinical_note(item):
    try:
        validated = ClinicalNoteContract(**item)
        return ValidationOk(validated)
    except ValidationError as e:
        return ValidationFail(e)
```

Each item: validated.

### 3.3 The "extra fields" handling

Strict (fail on unknown):

- Catches schema-creep early.
- Forces explicit changes.

Lenient (ignore unknown):

- More flexible.
- May miss meaningful changes.

Per contract.

### 3.4 The "missing required field" handling

Per field:

- Required = must be present.
- Optional = may be absent.

Validation per field.

### 3.5 The schema versioning

Schemas change:

- v3.0 → v3.1 (additive; backwards-compatible).
- v3.0 → v4.0 (breaking).

Consumers pin to a version.

Cross-link to [ai-architecture-reference-architecture / context-and-prompt-architecture / prompt-as-api-discipline.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/context-and-prompt-architecture/prompt-as-api-discipline.md) for semver discipline.

### 3.6 The breaking-change deprecation

Same lifecycle as APIs:

- Announce.
- Coexistence.
- Default switch.
- Removal.

Cross-link.

### 3.7 The "we got an unexpected field" detection

If extra fields appear:

- Alert.
- Investigate (supplier change?).
- Update contract if intentional.

Drift detection.

### 3.8 The schema documentation

For consumers:

- Documentation generated from schema.
- Example records.
- Field meanings.

Discoverable.

### 3.9 The schema-as-contract enforcement

CI:

- Consumers test against schema mock.
- Production: validation per item.
- Both verify contract.

---

## 4. Freshness contracts

The timing agreement.

### 4.1 Per-source freshness SLA

```yaml
clinical-notes-freshness:
  max_staleness_p99: 24 hours
  expected_average_staleness: 12 hours
  delivery_window: 02:00-04:00 UTC nightly
  
formulary-freshness:
  max_staleness_p99: 1 week
  update_frequency: weekly
  delivery_window: Sundays 01:00 UTC

guidelines-freshness:
  max_staleness_p99: 1 month
  update_frequency: monthly
  delivery_window: 1st of month 00:00 UTC
```

Per source.

### 4.2 Freshness measurement

Per item:

- updated_at (from source).
- ingested_at (at corpus).
- staleness = ingest - update.

Per-source aggregate.

### 4.3 The freshness-violation alerting

If staleness > SLA:

- Alert.
- Diagnose (connector? source? schema change?).
- Resolve.

Cross-link to [retrieval-corpus-engineering.md §4](./retrieval-corpus-engineering.md).

### 4.4 The "we promise" vs "we deliver" gap

If actual freshness < promised:

- Communicate.
- Either change promise (lower SLA).
- Or increase delivery cadence.

Honest.

### 4.5 The cross-source dependency

If multiple sources:

- Each has its own freshness SLA.
- Combined output freshness = worst source.

Manage per-source.

### 4.6 The provider-side delays

Supplier may delay:

- Source unavailable.
- Approval workflows.

Track per-supplier; communicate.

### 4.7 The "freshness recovery" workflow

When SLA violated:

- Catch up.
- Verify return to SLA.

Documented.

### 4.8 The "we don't measure freshness" gap

Without measurement:

- SLA aspirational, not enforceable.

Measure per §4.2.

### 4.9 The freshness-vs-cost trade-off

Higher freshness = higher infrastructure cost.

Per source: what's worth it.

---

## 5. Content-type contracts

Format and quality agreements.

### 5.1 Content guarantees

Per field / content:

- Encoding: UTF-8.
- HTML stripped (for text fields).
- No PII outside designated fields.
- Maximum length: 50,000 characters.
- Etc.

Per content type.

### 5.2 PII guarantees

For PII-bearing data:

- Specific fields contain PII (designated).
- Other fields don't.
- Verification.

Cross-link to [ai-architecture-reference-architecture / multi-tenancy-and-isolation / per-tenant-prompt-and-context.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-prompt-and-context.md).

### 5.3 The "content-type drift" detection

Detect:

- New patterns in data.
- Unexpected characters.
- New fields appearing in metadata.

Alert.

### 5.4 The encoding-anomaly detection

For text:

- Mojibake patterns.
- Unicode normalization issues.

Detect; flag.

### 5.5 The "supplier sent unexpected content" handling

If content violates contract:

- Alert.
- Quarantine the affected items.
- Investigate.
- Resolve.

### 5.6 The content-quality SLA

Per contract:

- 99.9% of items meet contract.
- Quarantine the 0.1%.
- Address.

Per-source.

### 5.7 The text-cleaning contract

Some contracts include cleaning:

- Supplier delivers clean text.
- Or consumer cleans.

Per contract.

### 5.8 The structure-preserved content

For structured fields:

- Schema (key-value).
- JSON / structured.

Validation enforces.

### 5.9 The "schema for content within a field" sub-schemas

For embedded structures:

- Sub-schema for nested fields.
- Recursive validation.

---

## 6. Change-notification protocol

How suppliers communicate changes.

### 6.1 The notification window

```yaml
change-notification:
  patch (clarifications): immediate (no advance required)
  minor (additive): 30 days advance notice
  major (breaking): 60-90 days advance notice
```

Per change type.

### 6.2 The notification channels

- Email to consumers.
- Slack channel.
- Wiki update.
- Release notes.

Multi-channel.

### 6.3 The acknowledgment requirement

For major changes:

- Consumer acknowledges.
- Migration plan documented.
- Timeline committed.

For minor: notification only.

### 6.4 The "we found out about the schema change in production" failure

If notification missed:

- Production breaks.
- Recovery improvised.

Mitigation:

- Schema-version detection at ingestion.
- Backups: drift alerts.

### 6.5 The supplier-deprecation handling

When a supplier deprecates data:

- Notification with timeline.
- Migration path.
- Coexistence period.

### 6.6 The cross-team coordination

For internal suppliers (other teams):

- Slack / email + Wiki.
- Regular sync meetings.

For external suppliers:

- Email + Wiki + release notes.

Per supplier.

### 6.7 The "we have many suppliers; can't track all" reality

Catalog:

- Per supplier.
- Notification status.
- Acknowledged changes.

Tracked.

### 6.8 The contract-renegotiation

Periodically:

- Reassess SLAs.
- Adjust based on reality.
- Re-document.

Yearly.

### 6.9 The "we discovered the change after deployment" detection

Drift detection (cross-link to §3.7):

- Catch schema changes upstream.
- Alert; investigate.

---

## 7. Contract violation handling

When contracts are violated.

### 7.1 The violation classification

- **Severity 0 (Critical).** Data integrity affected; immediate response.
- **Severity 1 (High).** Quality affected; same-day response.
- **Severity 2 (Medium).** Process issue; within-week response.
- **Severity 3 (Low).** Documentation issue; routine update.

Per violation.

### 7.2 The violation alerting

Per severity:

- S0: page.
- S1: page (business hours) or ticket.
- S2-S3: ticket.

Routing per severity.

### 7.3 The mitigation workflow

For violations:

- Quarantine affected data.
- Notify supplier.
- Diagnose.
- Remediate.

Documented.

### 7.4 The supplier-engagement

For supplier-side violations:

- Communicate.
- Negotiate fix.
- Track to completion.

Relationship management.

### 7.5 The "we'll work around the violation" anti-pattern

Building consumer-side workarounds for supplier violations:

- Slows the consumer.
- Hides the actual violation.
- Supplier never fixes.

Push back; escalate.

### 7.6 The contract-breach consequences

Repeated supplier breaches:

- Document.
- Escalate (technical leadership; or commercial if external).
- Consider alternative supplier.

Accountability.

### 7.7 The "we accept the violation" recognition

Sometimes violations are accepted:

- Documented.
- Workaround agreed.
- Compensating control.

But: deliberate, not accidental.

### 7.8 The post-violation review

Per significant violation:

- Why did it happen?
- Was notification missed?
- Was monitoring insufficient?
- Improve.

Lifecycle.

### 7.9 The violation log

Per violation:

```yaml
violation:
  date: 2026-05-15
  supplier: EHR team
  contract: ehr-clinical-notes-v3.2
  severity: S1
  description: encoding inconsistency in note_text (Mojibake in 2% of records)
  detection: drift alert
  mitigation: quarantine affected records; supplier re-export
  resolution: 2026-05-17
  prevention: encoding validation added to ingestion pipeline
```

Documented.

---

## 8. Worked Meridian example

Meridian's data-contract practice.

### 8.1 The contract catalog

```
clinical-notes-contract (v3.2):
  Supplier: EHR team
  Consumer: AI platform
  Status: in production

formulary-contract (v5.1):
  Supplier: Pharmacy team
  Consumer: AI platform
  Status: in production

guidelines-contract (v4.0):
  Supplier: Clinical Informatics team
  Consumer: AI platform
  Status: in production

customer-data-contract (per customer):
  Supplier: customer integration
  Consumer: AI platform
  Status: per-customer; versioned

internal-policy-contract (v2.3):
  Supplier: IT team
  Consumer: AI platform
  Status: in production
```

5 active contracts; tracked.

### 8.2 The schema validation pipeline

For each source:

```
Source → Ingestion service →
  Schema validation →
    Pass: process; load to corpus.
    Fail: quarantine; alert.
```

Per-item validation.

Validation rate: ~99.7% pass; 0.3% quarantine (varies).

### 8.3 The Q1 2026 schema change scare

EHR team made a schema change:

- New optional field added to clinical notes.
- Notification: 5 days advance via Slack.
- Consumer: AI platform.

Acceptable change; minor version bump (v3.1 → v3.2).

Updated consumer; tested; deployed.

### 8.4 The Q2 2026 schema break

Formulary team made unexpected change:

- Schema for medication-id changed format (no notification).
- Production pipeline broke.

Detection:

- Drift alert fired on first failed batch.
- Recovery: schema updated; consumer-side parser updated; backfill.

Lesson:

- Notification process tightened.
- Formulary team's process documented.

### 8.5 The Q3 2026 supplier-deprecation

External research database supplier deprecated their API:

- 90-day notice.
- Acknowledged.
- Migrated to new API.

Smooth.

### 8.6 The freshness SLA tracking

Per source dashboard:

```
EHR clinical notes: P99 staleness 18h (SLA <24h) ✓
Formulary: P99 staleness 4 days (SLA <1 week) ✓
Guidelines: P99 staleness 3 weeks (SLA <1 month) ✓
Customer data: per-customer; mostly within SLA
Internal policies: P99 staleness 6 days (SLA <2 weeks) ✓
```

Reviewed monthly.

### 8.7 The contract repository

Stored:

- /contracts/ folder in version control.
- Per contract: spec, schema, SLAs.
- Versioned.

Catalog.

### 8.8 The supplier-relationship cadence

- Weekly: ad-hoc issues.
- Monthly: per-supplier sync (large suppliers).
- Quarterly: contract review across suppliers.
- Annual: re-negotiation.

Active.

### 8.9 The infrastructure cost

- Schema validation: ~$200/month.
- Drift detection: ~$300/month.
- Supplier management time: ~3 hours/week.

Total: modest.

### 8.10 The Q1 2026 customer-data contract incident

A new customer onboarded with their own data feed:

- Contract not formalized.
- Their data had encoding issues.
- AI quality affected (~5% of their cases).

Recovery:

- Formal contract negotiated.
- Encoding validation added.
- Issue resolved.

Lesson: customer-data contracts now required at onboarding.

### 8.11 The lessons

- Contracts catch issues before they break things.
- Notification protocol matters; suppliers need engagement.
- Validation at ingestion is essential.
- Customer-data contracts are critical for multi-tenant platforms.

---

## 9. Anti-patterns

### 9.1 The contract-not-documented

**Pattern.** Implicit understanding; no written contract.

**Corrective.** Document per §2.

### 9.2 The supplier-changes-without-notification

**Pattern.** Schema changes; pipeline breaks; nobody warned.

**Corrective.** Notification protocol per §6.

### 9.3 The validation-not-at-ingestion

**Pattern.** Bad data flows to corpus; quality degrades silently.

**Corrective.** Per-item validation per §3.2.

### 9.4 The "we accept everything; figure it out later" lenient

**Pattern.** No schema strictness; everything accepted.

**Corrective.** Strict validation per §3.3.

### 9.5 The freshness-SLA-unmonitored

**Pattern.** SLA exists; not measured; staleness drift.

**Corrective.** Measurement per §4.2.

### 9.6 The cross-team breakage cycle

**Pattern.** Supplier breaks contract; consumer fixes; same supplier breaks again next time.

**Corrective.** Supplier engagement + accountability per §7.6.

### 9.7 The customer-data-without-contract

**Pattern.** Customer data ingested without formal contract.

**Corrective.** Contract at onboarding per §8.10.

### 9.8 The encoding-not-validated

**Pattern.** Mojibake / encoding issues silently in corpus.

**Corrective.** Encoding validation per §5.4.

### 9.9 The "we found a violation; never logged" lack of audit

**Pattern.** Violations occur; not documented; pattern invisible.

**Corrective.** Violation log per §7.9.

### 9.10 The contract-vs-reality drift

**Pattern.** Contract documented; reality has shifted; contract not updated.

**Corrective.** Periodic review per §6.8.

---

## 10. Findings (sprint-assignable)

### DATA-DC-001 — Severity: Critical
**Finding.** Data sources without contracts.
**Recommendation.** Document per §2.
**Owner.** data engineering + AI platform, sprint N+1.

### DATA-DC-002 — Severity: Critical
**Finding.** Schema validation not at ingestion.
**Recommendation.** Per §3.2.
**Owner.** data engineering, sprint N+1.

### DATA-DC-003 — Severity: Critical
**Finding.** Freshness SLAs unmonitored.
**Recommendation.** Per §4.
**Owner.** data engineering + observability, sprint N+1.

### DATA-DC-004 — Severity: High
**Finding.** Change-notification protocol absent.
**Recommendation.** Per §6.
**Owner.** AI platform + supplier teams, sprint N+2.

### DATA-DC-005 — Severity: High
**Finding.** Drift detection absent for schema changes.
**Recommendation.** Per §3.7.
**Owner.** data engineering, sprint N+2.

### DATA-DC-006 — Severity: High
**Finding.** Contract violation handling undefined.
**Recommendation.** Per §7.
**Owner.** AI platform + data engineering, sprint N+2.

### DATA-DC-007 — Severity: High
**Finding.** Customer data without formal contract.
**Recommendation.** Per §8.10.
**Owner.** AI platform + product, sprint N+2.

### DATA-DC-008 — Severity: Medium
**Finding.** Schema versioning absent.
**Recommendation.** Per §3.5.
**Owner.** data engineering, sprint N+3.

### DATA-DC-009 — Severity: Medium
**Finding.** Contract catalog absent.
**Recommendation.** Per §2.9.
**Owner.** data engineering, sprint N+3.

### DATA-DC-010 — Severity: Medium
**Finding.** Content-type validation absent.
**Recommendation.** Per §5.
**Owner.** data engineering, sprint N+3.

### DATA-DC-011 — Severity: Medium
**Finding.** Volume expectations not tracked.
**Recommendation.** Per §2.4.
**Owner.** data engineering, sprint N+3.

### DATA-DC-012 — Severity: Medium
**Finding.** Supplier engagement informal.
**Recommendation.** Cadence per §8.8.
**Owner.** AI platform, sprint N+3.

### DATA-DC-013 — Severity: Medium
**Finding.** Violation log absent.
**Recommendation.** Per §7.9.
**Owner.** data engineering, sprint N+4.

### DATA-DC-014 — Severity: Medium
**Finding.** Contract renegotiation cadence absent.
**Recommendation.** Annual per §6.8.
**Owner.** AI platform + supplier teams, sprint N+4.

### DATA-DC-015 — Severity: Low
**Finding.** PII handling guarantees not in contracts.
**Recommendation.** Per §5.2.
**Owner.** AI platform + privacy, sprint N+5.

### DATA-DC-016 — Severity: Low
**Finding.** Cross-source dependencies not documented.
**Recommendation.** Per §4.5.
**Owner.** data engineering, sprint N+5.

### DATA-DC-017 — Severity: Low
**Finding.** Contract documentation lacks examples.
**Recommendation.** Per §3.8.
**Owner.** data engineering, sprint N+6.

### DATA-DC-018 — Severity: Low
**Finding.** Cross-team supplier accountability mechanisms absent.
**Recommendation.** Per §7.6.
**Owner.** engineering management, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Catalog all data sources per §2.9.**
- [ ] **Document contracts per source per §2.**
- [ ] **Schema validation per source per §3.2.**
- [ ] **Freshness monitoring per source per §4.**
- [ ] **Drift detection per §3.7.**
- [ ] **Change-notification protocol per §6.**
- [ ] **Violation log per §7.9.**
- [ ] **Supplier-engagement cadence per §8.8.**
- [ ] **Customer-data contract at onboarding per §8.10.**
- [ ] **Annual contract review per §6.8.**

---

## 12. References

**In this folder.**
- [retrieval-corpus-engineering.md](./retrieval-corpus-engineering.md) — corpus as consumer.
- [dataset-versioning.md](./dataset-versioning.md) — versioning includes contracts.
- [data-quality-for-ai.md](./data-quality-for-ai.md) — quality.
- [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md) — contracts prevent contamination.
- [training-eval-split-discipline.md](./training-eval-split-discipline.md) — split discipline.
- [labeling-and-annotation.md](./labeling-and-annotation.md) — labeling contracts.
- [synthetic-data-generation.md](./synthetic-data-generation.md) — synthetic generation contracts.

**Elsewhere in this repo.**
- [observability-and-telemetry/](../observability-and-telemetry/) — observability for contract violations.
- [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md) — contract violations as incidents.

**Sibling repos.**
- [ai-architecture-reference-architecture / data-architecture-for-ai / data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md) — architectural data contracts.
- [ai-architecture-reference-architecture / integration-architecture/integration-failure-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/integration-failure-patterns.md) — integration failures.

**External.**
- Data contracts manifesto (Chad Sanderson).
- Schema-as-code patterns (Avro, JSON Schema, Protobuf).
- Open Data Contract Standard (ODCS).
- Stripe / Twilio API versioning as reference.
