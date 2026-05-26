# Agent Anti-Patterns

> **Audience.** Tech leads auditing an existing agent feature. Engineers about to start their first agent who want to know what to avoid. Anyone whose answer to "why is our agent in this state?" should be a specific named pattern, not a vague gesture. **Scope.** The eight agent engineering anti-patterns observed most often in 2026 production — what they look like, why they happen, what they cause, and the corrective. A consolidated catalogue across the agent-engineering folder; references the per-topic depths for the corrective patterns. **Worked client.** Meridian Health.

---

## 1. Why this document exists

Every doc in this folder has an "anti-patterns" section covering the patterns specific to that topic — loop design anti-patterns in [agent-loop-design.md](./agent-loop-design.md), tool architecture anti-patterns in [tool-architecture.md](./tool-architecture.md), and so on. This document is the consolidated catalogue of the eight patterns that appear most often across teams — patterns that span topics, that recur regardless of the team's framework choice, and that produce the agent-feature incidents that recur in the post-incident reviews.

The catalogue is operational, not theoretical. Each anti-pattern in this document has been observed in production at multiple companies; each has caused at least one notable incident class; each has a corrective that the team-level investments described elsewhere in this folder implement. The patterns aren't novel research findings — they're the things that go wrong when teams ship agents without the engineering disciplines this folder describes.

Two things to keep in mind while reading:

First, anti-patterns rarely appear alone. A team running into one is usually running into three. "Agent for everything" tends to come with "no turn budget" and "no trajectory observability" because the same conditions (the team didn't yet take agent operations seriously) produce all three. Audits should look for clusters.

Second, the corrective is rarely a code change in isolation. The patterns reflect engineering practice and team disciplines. The corrective is typically a combination of code, prompts, runbooks, eval coverage, and on-call practices. The references to the per-topic docs are where the corrective's depth lives.

Each section below follows the same structure: **what it looks like**, **why it happens**, **what it causes**, **the corrective**. The eight together are the operational vocabulary for agent engineering quality.

Structure: (2) "agent for everything"; (3) "no turn budget"; (4) "no cost budget"; (5) "tools with side effects without idempotency"; (6) "memory is actually context stuffing"; (7) "retry everything on error"; (8) "no trajectory observability"; (9) "human-in-the-loop as rubber stamp"; (10) worked Meridian example (audit and remediation); (11) findings; (12) adoption checklist (the agent quality audit); (13) references.

---

## 2. Anti-pattern #1: "Agent for everything"

### 2.1 What it looks like

Every AI feature in the team's roadmap is built as an agent. The feature could have been:

- A single LLM call (input → response with structured output).
- A workflow with three LLM steps.
- A hybrid with a small agent step inside a workflow.

But it was built as a top-level agent loop. The team's reasoning: "agents are more general; they handle more cases; the framework supports it; we already know agents from the previous feature."

### 2.2 Why it happens

- The team's first AI feature was an agent (often because the framework demo was an agent demo); subsequent features default to the same shape without re-examining.
- Framework marketing positions multi-step agentic workflows as the headline feature.
- "Agent" feels more advanced than "workflow"; the team perceives it as the modern shape.
- The team hasn't built the discipline of explicit shape selection (per [agent-vs-workflow-decision.md](./agent-vs-workflow-decision.md)).

### 2.3 What it causes

- 3–10× the operational cost of the equivalent workflow.
- Latency 3–5× the equivalent workflow.
- A debugging surface that requires trajectory reading even for simple cases that could have been a single LLM call's input + output.
- Eval that is harder and more expensive than necessary.
- On-call burden disproportionate to the feature's importance.

### 2.4 The corrective

Apply the shape decision tree per [agent-vs-workflow-decision.md](./agent-vs-workflow-decision.md) to every AI feature. The default should be workflow; agent only when the criteria justify. Most features that were built as agents will reveal, on examination, that they should be workflows or hybrids.

Refactor candidates: features whose production traces show the agent following the same path on 80%+ of requests. The recurring pattern is workflow-shaped; refactor.

The refactoring is usually a single-sprint effort and pays for itself in cost reduction within months.

---

## 3. Anti-pattern #2: "No turn budget"

### 3.1 What it looks like

The agent's loop has no maximum turn count, or the maximum is so high (e.g., 200) that it might as well be unbounded. The team's reasoning: "we don't want to limit the agent; some tasks need many turns; we'll trust the model to stop."

### 3.2 Why it happens

- The team didn't engineer the runner with budgets; the loop is `while not done`.
- The team set a high limit "just in case" and never tightened it.
- The team relied on the cost budget as the only stop; turn budget was deemed redundant.
- The team assumed the model would always reach a final answer; "the model is smart enough."

### 3.3 What it causes

- Runaway loops on edge-case inputs. The agent doesn't reach termination and continues turn after turn.
- Cost incidents — even with a cost budget as a backstop, the per-turn cost variance means turn-budget enforcement is often the first stop.
- Latency outliers in the hundreds of seconds for unusual cases.
- User-visible "this is taking forever" experiences that erode trust.

### 3.4 The corrective

Per [agent-loop-design.md](./agent-loop-design.md) section 5: turn budget set to a calibrated value (typically 5–20 for most agents, 50 for unusual high-complexity cases). Enforced at the runner before each new turn. Breach triggers the final-turn pattern (one last LLM call with a tight cost cap, instructed to produce a graceful failure response).

The turn budget is part of the four-budget enforcement (turn / cost / time / tool-call). Implementing one without the others leaves gaps; implementing all four is the discipline.

---

## 4. Anti-pattern #3: "No cost budget"

### 4.1 What it looks like

Per-request cost has no enforced ceiling. The team relies on "we'll alert if cost goes up" — alerting, not enforcement. Or the team has a budget written in a planning document that nothing enforces.

### 4.2 Why it happens

- The team didn't wire cost tracking at request time; cost is observed only post-hoc via the provider's invoice.
- The team has alerts on cost but no automated stop.
- The team trusts the turn budget alone to bound cost; the trust is misplaced because per-turn cost varies widely.
- The team's first cost incident hasn't happened yet; the discipline of cost enforcement hasn't been forced.

### 4.3 What it causes

- The notable cost incidents (per [agent-cost-control.md](./agent-cost-control.md) section 1) — $50,000+ bills from runaway agents over a weekend.
- Cost surprises at month-end when the invoice arrives.
- An adversarial-misuse exposure — a user can intentionally drain the team's budget if they discover how.
- Hard conversations with finance and product about cost ownership.

### 4.4 The corrective

Per [agent-cost-control.md](./agent-cost-control.md):

- Per-request budget enforced at the runner.
- Per-tenant caps enforced at the gateway.
- Per-feature caps as backstop.
- Cost attribution per [cost-attribution.md](../cost-and-finops/cost-attribution.md) so incidents are triageable.
- Tier routing (cheap orchestrator + expensive specialist) to reduce baseline cost.
- Runbook with sub-15-minute mean-time-to-mitigation target.
- Authority-to-act for on-call.

The discipline is comprehensive because cost incidents compound by the minute. Half-measures (alerting only, single-layer budgets) leave gaps that compound.

---

## 5. Anti-pattern #4: "Tools with side effects without idempotency"

### 5.1 What it looks like

The agent has tools that send messages, create records, charge cards, or otherwise affect the real world. The tools have no idempotency key. The runner has bounded retries (per anti-pattern #6). On transient failure, the runner retries the same tool call.

The patient receives two SMS messages. The customer's card is charged twice. The record is created in duplicate.

### 5.2 Why it happens

- The tool was built without idempotency thinking; the tool's author was focused on the success path.
- The retry behaviour was wired without coordinating with the tool's side-effect characteristics.
- The team's tools predate the agent integration; they were designed for a different invocation model.
- The team didn't engineer the side-effect taxonomy (per [tool-architecture.md](./tool-architecture.md) section 6.1).

### 5.3 What it causes

- Duplicate side effects in production. Each is its own small incident; customer-facing apologies; cleanup work.
- Cumulative reputation damage as duplicates accumulate.
- For high-stakes side effects (financial, medical), real-world impact beyond the technical fix.
- Loss of trust in the agent's reliability that lingers after the technical fix.

### 5.4 The corrective

Per [tool-architecture.md](./tool-architecture.md) section 6:

- Side-effect taxonomy: read-only / idempotent / non-idempotent-with-key / non-idempotent.
- Idempotency keys required on non-idempotent-with-key tools; runtime de-duplicates.
- Approval gate (propose + execute) for non-idempotent high-stakes tools.
- Audit of every existing side-effecting tool; classification recorded; gaps fixed.

The audit is the most important step for existing systems. It surfaces tools the team didn't realise had side effects (a "lookup" that actually writes an analytics event; a "fetch" that triggers a downstream notification). Each surfaced tool gets the appropriate corrective.

---

## 6. Anti-pattern #5: "Memory is actually context stuffing"

### 6.1 What it looks like

The team enabled the framework's "memory" feature, which writes every conversation turn to a vector store. On each invocation, it retrieves and injects the top-N most-similar prior turns into context.

The model receives an increasingly noisy context window of past turns. Quality degrades; cost rises. The team calls this "memory."

### 6.2 Why it happens

- The framework's memory feature is presented as a single switch; the team enabled it without analysis.
- "Memory" was a vague requirement on the roadmap; the team interpreted it as "the model should remember prior turns" and reached for whatever the framework called "memory."
- The team didn't decompose memory into types (short-term / long-term / episodic / semantic per [memory-engineering.md](./memory-engineering.md)).
- The team didn't measure context-window quality; the cost rise was attributed to "model getting more complex" rather than to bad memory.

### 6.3 What it causes

- Cost rises (larger context per call); quality drops (noisy context).
- Hallucinated cross-conversation facts (the retrieved turns are from other users or sessions and contaminate this one).
- Privacy concerns when retrieval crosses appropriate scopes.
- An expensive infrastructure (vector store, embedding pipeline) that doesn't provide proportional value.

### 6.4 The corrective

Per [memory-engineering.md](./memory-engineering.md):

- Decompose the use case into memory types; build only the types needed.
- Short-term context engineered explicitly (running summary, structured extraction); not "all of history retrieved as similar turns."
- Long-term memory as RAG over a curated corpus; not as auto-write of conversation.
- Episodic memory per-conversation with explicit scoping; not cross-conversation similarity.
- Semantic memory with deliberate writes; not auto-write of every turn.

The corrective often *simplifies* the system: the vector store goes away (or its purpose narrows), the cost drops, the quality improves.

---

## 7. Anti-pattern #6: "Retry everything on error"

### 7.1 What it looks like

The agent (or the runner) encounters any error and retries. The retries are unbounded, or so high (10+) that they may as well be unbounded. The agent re-tries failing tools, the runner retries failing LLM calls, the agent tries the same approach repeatedly hoping for a different result.

### 7.2 Why it happens

- The team didn't classify errors as transient vs permanent (per [error-and-partial-failure.md](./error-and-partial-failure.md) section 3.1).
- The retry logic was added defensively without bounding; "more retries = more robust."
- The runner's retry policy doesn't distinguish: any failure → retry.
- The agent's prompt instructs "if a tool fails, try again" without conditions.

### 7.3 What it causes

- Cost incidents from repair loops. The agent tries to make a wrong argument right; each retry is another LLM call.
- Latency spikes from cascading retries on transient infrastructure issues.
- Permanent-failure cases that should have terminated immediately consume budget instead.
- Confused agents — the agent attempts the same call many times; the trajectory becomes hard to read.

### 7.4 The corrective

Per [error-and-partial-failure.md](./error-and-partial-failure.md):

- Transient vs permanent classification on every tool error. Structured error envelope per [tool-architecture.md](./tool-architecture.md) section 5.2.
- Bounded retry policy at the runner (typically 3 attempts with exponential backoff for transient).
- Agent prompt with explicit guidance: do not re-retry a permanent failure; adapt the approach or escalate.
- Output validators at tool boundaries detect junk so repair loops don't run on inherent garbage.

The retry discipline is foundational; without it, every other corrective is undermined.

---

## 8. Anti-pattern #7: "No trajectory observability"

### 8.1 What it looks like

When the agent misbehaves, the team has no way to see why. There's no trace; only logs scattered across components. The on-call engineer reads the conversation log, reads the application logs, reads the model provider's logs (if they're available), and tries to piece together what happened. The investigation takes hours.

The next incident, the engineer guesses based on the prior pattern. The investigation isn't getting faster; the team isn't learning systematically.

### 8.2 Why it happens

- The team treated observability as something to add later; "later" arrived as the first incident.
- The team has logs (because the framework defaulted to them) but no structured trace.
- The team's observability investment went to metrics and dashboards (the easier part) without trajectory capture.
- The team underestimated agent-specific observability needs; treated agents as if they were single-call features.

### 8.3 What it causes

- Incident MTTM measured in hours instead of minutes.
- Recurring incidents because root causes aren't reliably identified.
- On-call rotation that engineers dread; agents become "the on-call feature."
- Investment in agent improvements stalls because the team can't measure or diagnose changes.

### 8.4 The corrective

Per [agent-observability.md](./agent-observability.md):

- Structured trace with one trace per request, hierarchical spans, attributes per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md).
- Vendor tool integration (LangSmith, Braintrust, Phoenix, Honeycomb, Datadog) or OpenTelemetry-based self-hosted.
- Per-feature dashboards with loop-health panels.
- Loop-aware alerts (not just generic error rate).
- Runbook with alert-to-trace links.
- Tail-based sampling for critical features so interesting cases are always captured.

The investment pays for itself within months: incidents diagnosed in minutes; recurring patterns identified and fixed; on-call burden manageable.

---

## 9. Anti-pattern #8: "Human-in-the-loop as rubber stamp"

### 9.1 What it looks like

The agent proposes a high-stakes action; a human review step exists; the human approves 99%+ of proposals without meaningful scrutiny. The HITL exists procedurally but doesn't add the quality bar it was designed to provide.

Variants:

- The reviewer is the same person who would have taken the action anyway; the review is theater.
- The reviewer is overloaded (1000+ approvals per day); approval is reflexive.
- The proposal lacks the context the reviewer needs to evaluate; "looks fine" is the default.
- The system's UI optimises for fast approval (a single button); the friction to reject is high.

### 9.2 Why it happens

- The HITL was added to satisfy a compliance or safety requirement; the requirement was met procedurally without commitment to the spirit.
- The reviewer's role and workload weren't designed; the HITL was bolted on.
- The proposals' UX prioritises agent throughput over review quality.
- The system doesn't measure rejection rates; if it did, the team would notice the rate was near zero.
- The team trusts the agent's quality; HITL is the legacy of a less-trustworthy version.

### 9.3 What it causes

- Real false positives reach production despite the HITL — the rubber stamp didn't catch them.
- Compliance fiction — auditors see the HITL in the design; the actual oversight is minimal; an incident reveals the gap publicly.
- Reviewer burnout from a role designed for thoughtful review but operated as throughput.
- Loss of the agent's quality signal — the team can't distinguish "agent is good" from "reviewer rubber-stamps everything."

### 9.4 The corrective

Restructure the HITL to add real value:

- **Reduce volume.** Auto-approve cases where rule-based or LLM-based confidence is high; route only uncertain cases to humans. The HITL catches the cases that genuinely need it.
- **Surface context to reviewer.** Per-proposal context (the agent's reasoning, the relevant inputs, the alternatives considered) so the reviewer can evaluate meaningfully.
- **Measure rejection rate.** A HITL with < 1% rejection rate is rubber-stamping; investigate.
- **Random-sample audit.** Even when most cases are auto-approved, random sampling for human review catches drift.
- **Reviewer training and role design.** The reviewer's role is reviewer, not throughput operator; SLA reflects review quality, not approval rate.
- **Time-budget per proposal.** Designed for thoughtful review; if proposals are arriving faster than the budget allows, the auto-approval logic needs to absorb more.

Sometimes the right corrective is to remove the HITL. If the agent's quality is high enough that the HITL adds nothing, accept the agent's behaviour and remove the procedure. The HITL is meaningful when the human's judgment changes outcomes; it's theater when it doesn't.

---

## 10. Worked Meridian example

Meridian's care-coordinator feature has avoided most of these anti-patterns. The team's audit cadence and the specific corrective for one anti-pattern it did encounter are illustrative.

### 10.1 The audit cadence

Quarterly anti-pattern audit. The team walks through each of the eight; checks whether the system exhibits any. Outcomes are recorded in the feature's quality document.

Findings over 14 months:

- **#1 ("agent for everything")** — never triggered. The feature was deliberately built as a hybrid (per [agent-vs-workflow-decision.md](./agent-vs-workflow-decision.md) section 9). Audit confirms the shape is still right.
- **#2 ("no turn budget")** — never triggered. Budgets in place from launch.
- **#3 ("no cost budget")** — never triggered. Cost discipline central from launch (per [agent-cost-control.md](./agent-cost-control.md)).
- **#4 ("tools without idempotency")** — *triggered once*. See section 10.2.
- **#5 ("memory as context stuffing")** — never triggered. Memory architecture deliberate (per [memory-engineering.md](./memory-engineering.md) section 9).
- **#6 ("retry everything")** — never triggered.
- **#7 ("no trajectory observability")** — never triggered.
- **#8 ("HITL as rubber stamp")** — partially triggered, see section 10.3.

### 10.2 Anti-pattern #4 detection and remediation

Q2-25 audit identified that the `log_action_in_ehr` tool (one of the care-coordinator's tools) lacked an idempotency key. The tool wrote audit entries to the EHR; retries on transient failure had produced duplicate audit entries in production.

The duplicate rate was 0.3% of audit writes — small but present. No patient-facing impact (audit duplicates were a backend nuisance), but the principle warranted the fix.

Corrective:

1. Add `idempotency_key` argument to the tool's schema (required).
2. Runtime de-duplication at the EHR adapter (de-dups within 24-hour window).
3. Audit-tool eval cases: deliberately retry a call with same key; verify single execution.
4. Backfill cleanup of historical duplicates (deterministic SQL fix).

Effort: ~1 sprint. Duplicate rate dropped to 0%. The audit caught the pattern before it became a notable incident.

### 10.3 Anti-pattern #8 partial trigger and corrective

Q4-25 audit identified that the `propose_followup_appointment` tool's HITL was approaching rubber-stamp territory. Care managers approved 97% of proposals; rejection rate had drifted from 8% (at launch) to 3%.

Investigation:

- The agent's proposal quality had genuinely improved (lower rejection appropriate).
- The reviewer UX had also been "optimised" — single-click approval; reject required typing a reason.
- Random-sample audit of recent approvals identified 4% that should have been rejected (the reviewer hadn't looked carefully).

So the system was simultaneously trustworthy (agent quality up) and untrustworthy (reviewer attention down).

Corrective:

1. Reviewer UX: single-click approval *and* single-click rejection (with optional reason).
2. Auto-approve high-confidence proposals (confidence model based on prior approval patterns); reviewer only sees uncertain proposals.
3. Random-sample audit weekly; results reported to reviewer team.
4. Reviewer training refresh on what to look for.

Outcome over 3 months: reviewer volume halved (auto-approval handled most cases); rejection rate on routed cases rose from 3% to 9% (reviewer attention restored on the cases that need it); audit-found errors dropped to <1%.

### 10.4 The audit's value

Two corrective actions in 14 months. Neither was triggered by a notable incident; both were caught by the audit cadence. The cost of the audits (~half a day each quarter) is recovered many times over by the incidents avoided.

The audit format also serves as a teaching tool — new engineers reading the audit learn the eight anti-patterns and the team's defenses against them. The catalogue is operational documentation.

### 10.5 The patterns that almost triggered

Audit also captures *near* triggers — the patterns that were considered and rejected:

- A product proposal to "let the agent draft outbound patient communications and a care manager approves" was reviewed; the team flagged risk of #8 (rubber-stamp), engineered the auto-approval / random-audit pattern, and shipped with the controls in place.
- A proposed tool that would have called a payment-processing API was reviewed; the team designed the propose+execute pattern (per [tool-architecture.md](./tool-architecture.md) section 6.4) before shipping.

The audit is not just a remediation tool; it's a design-review check.

---

## 11. Findings (sprint-assignable)

These findings are cross-cutting. Each maps to one or more anti-patterns above; the recommended actions reference the per-topic docs.

### AGT-AP-001 — Severity: Critical
**Finding.** Multiple AI features built as agents by default; no documented shape decision (anti-pattern #1).
**Recommendation.** Apply shape decision tree per [agent-vs-workflow-decision.md](./agent-vs-workflow-decision.md) to every feature; refactor candidates whose production behaviour is workflow-shaped.
**Owner.** ai-platform-eng + product, sprint N+1.

### AGT-AP-002 — Severity: Critical
**Finding.** Agent has no enforced turn or cost budgets (anti-patterns #2 + #3).
**Recommendation.** Implement four-budget runner per [agent-loop-design.md](./agent-loop-design.md) section 5; per-tenant caps per [agent-cost-control.md](./agent-cost-control.md).
**Owner.** ai-platform-eng, sprint N+1.

### AGT-AP-003 — Severity: Critical
**Finding.** Side-effecting tools lack idempotency keys (anti-pattern #4).
**Recommendation.** Audit per [tool-architecture.md](./tool-architecture.md) section 6; classify each tool; implement keys on non-idempotent-with-key; propose+execute for high-stakes.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-AP-004 — Severity: Critical
**Finding.** No trajectory observability; incident MTTR > 1 hour (anti-pattern #7).
**Recommendation.** Trace instrumentation per [agent-observability.md](./agent-observability.md); vendor or OTEL stack; loop-aware alerts.
**Owner.** ai-platform-eng + ops, sprint N+1.

### AGT-AP-005 — Severity: High
**Finding.** "Memory" implementation is auto-write of conversation turns to vector store; cost up, quality down (anti-pattern #5).
**Recommendation.** Memory taxonomy analysis per [memory-engineering.md](./memory-engineering.md); rebuild as appropriate types.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-AP-006 — Severity: High
**Finding.** Runner retries unbounded on any failure; cost incidents traceable to repair loops (anti-pattern #6).
**Recommendation.** Transient/permanent classification per [error-and-partial-failure.md](./error-and-partial-failure.md); bounded retries; agent prompt updated.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-AP-007 — Severity: High
**Finding.** HITL approval rate > 95% with no rejection-rate measurement (anti-pattern #8).
**Recommendation.** Per section 9.4: reduce volume via auto-approve; surface context to reviewer; measure rejection rate; random-sample audit.
**Owner.** ai-platform-eng + product + reviewer team, sprint N+2.

### AGT-AP-008 — Severity: High
**Finding.** Agent's worst-case cost is unknown; no per-tenant caps.
**Recommendation.** Cost decomposition per [agent-cost-control.md](./agent-cost-control.md); per-tenant caps; per-feature backstop.
**Owner.** ai-platform-eng + ops, sprint N+2.

### AGT-AP-009 — Severity: High
**Finding.** Quarterly anti-pattern audit not performed; quality issues compound undetected.
**Recommendation.** Audit cadence per section 10.1; checklist; recorded in feature quality doc.
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### AGT-AP-010 — Severity: Medium
**Finding.** Generic alerts only; loop-specific pathologies undetected (related to anti-pattern #7).
**Recommendation.** Loop-aware alerts per [agent-observability.md](./agent-observability.md) section 5.
**Owner.** ai-platform-eng + ops, sprint N+3.

### AGT-AP-011 — Severity: Medium
**Finding.** Tools-with-side-effects audit not on a schedule; new tools may introduce anti-pattern #4.
**Recommendation.** Tool addition checklist includes idempotency review; quarterly catalogue audit.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-AP-012 — Severity: Medium
**Finding.** Runbook missing or lacks per-anti-pattern guidance; incidents resolved by improvisation.
**Recommendation.** Runbook per [agent-observability.md](./agent-observability.md) section 6.
**Owner.** ops + ai-platform-eng, sprint N+3.

### AGT-AP-013 — Severity: Medium
**Finding.** New features ship without anti-pattern review; same patterns re-introduced.
**Recommendation.** Design review checklist includes the eight anti-patterns; reviewer signs off.
**Owner.** ai-platform-eng + tech-leads, sprint N+3.

### AGT-AP-014 — Severity: Medium
**Finding.** Engineers refer to anti-patterns vaguely ("the agent is doing the wrong thing"); shared vocabulary absent.
**Recommendation.** Anti-pattern names from this catalogue used in design discussions, PR reviews, and incident retros.
**Owner.** ai-platform-eng + tech-leads, sprint N+4.

### AGT-AP-015 — Severity: Low
**Finding.** Eval set doesn't cover anti-pattern scenarios; regressions toward anti-patterns invisible.
**Recommendation.** Per [agent-evals.md](./agent-evals.md): trajectory eval includes anti-pattern detection (loop, thrash, unnecessary repair).
**Owner.** ai-platform-eng + feature-team, sprint N+4.

### AGT-AP-016 — Severity: Low
**Finding.** Tool idempotency tests absent; idempotency claims unverified.
**Recommendation.** Idempotency tests per tool; CI gate.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-AP-017 — Severity: Low
**Finding.** Memory architecture decisions not documented; future engineers may re-introduce context-stuffing.
**Recommendation.** Memory design document per feature per [memory-engineering.md](./memory-engineering.md) section 9.
**Owner.** ai-platform-eng + feature-team, sprint N+5.

### AGT-AP-018 — Severity: Low
**Finding.** Multi-feature catalogue of anti-pattern audit results not maintained; cross-feature trends invisible.
**Recommendation.** Quarterly portfolio-level audit summary; trends fed to AI platform roadmap.
**Owner.** ai-platform-eng + product, sprint N+5.

---

## 12. Adoption sequencing checklist (the agent quality audit)

The audit checklist for an existing agent. Walk through each of the eight; record whether the system exhibits the pattern; for each "yes," follow the corrective.

### 12.1 The audit walkthrough

For each of the eight anti-patterns:

- [ ] **#1 "Agent for everything."** Read the feature's design document. Was the shape (agent / workflow / hybrid) chosen explicitly? Apply the decision tree per [agent-vs-workflow-decision.md](./agent-vs-workflow-decision.md). Outcome: shape confirmed or refactor scoped.
- [ ] **#2 "No turn budget."** Inspect runner config. Is there a turn budget? Is it calibrated (5–20 typical)? Is enforcement at the runner? Outcome: budgets present and enforced, or implementation scoped.
- [ ] **#3 "No cost budget."** Inspect runner config and gateway config. Per-request budget? Per-tenant caps? Per-feature cap? Attribution wired? Tier routing? Outcome: full cost discipline present, or remediation scoped.
- [ ] **#4 "Tools without idempotency."** Audit the tool catalogue. For each tool, what's its side-effect category? Are idempotency keys required where needed? Are high-stakes tools using propose+execute? Outcome: catalogue classified and gaps closed.
- [ ] **#5 "Memory as context stuffing."** Inspect memory implementation. Which of the four types (short-term, long-term, episodic, semantic) are used? Are they engineered per the taxonomy, or is it auto-write-everything? Outcome: memory architecture confirmed or rebuild scoped.
- [ ] **#6 "Retry everything."** Inspect retry policy. Bounded? Transient/permanent classification? Output validators? Agent prompt guidance? Outcome: retry discipline confirmed or remediation scoped.
- [ ] **#7 "No trajectory observability."** Inspect observability stack. Structured trace? Hierarchical spans? Loop-aware alerts? Runbook integration? Outcome: observability adequate or build-out scoped.
- [ ] **#8 "HITL as rubber stamp."** Inspect HITL workflow. Approval rate? Rejection rate trend? Reviewer UX? Random audit in place? Outcome: HITL effective or restructure scoped.

The walkthrough takes 2–4 hours per feature. Quarterly cadence.

### 12.2 The remediation sequencing

For each "yes," the remediation is scoped against the per-topic doc. Typical sequencing:

- Sprint N+1: critical-severity findings (cost budget, idempotency, observability).
- Sprint N+2: high-severity findings (memory, retry, HITL, per-tenant caps).
- Sprint N+3: medium-severity findings (shape refactor, runbook, alerts).
- Sprint N+4: low-severity findings (test coverage, documentation, eval coverage).

A team starting from a baseline of zero (none of the disciplines in place) can reach a healthy state in 2–3 quarters of focused work. The investment is recovered through reduced incident frequency and severity.

### 12.3 The new-feature checklist

For each new agent feature, the design review uses the same eight as a checklist. The reviewer signs off only when none of the patterns are present at design time.

### 12.4 The portfolio view

A team operating multiple agents maintains a portfolio-level summary:

| Feature | #1 | #2 | #3 | #4 | #5 | #6 | #7 | #8 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| care-coordinator | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ (after Q4 fix) |
| analytics-copilot | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | n/a |
| patient-summary | n/a (workflow) | n/a | ✓ | ✓ | n/a | ✓ | ✓ | n/a |

The portfolio surfaces cross-feature trends; the platform team can prioritise investments that close gaps across multiple features.

---

## 13. References

- [agent-engineering-playbook.md](./agent-engineering-playbook.md) — broader practice; section 11 (anti-patterns overview).
- [agent-vs-workflow-decision.md](./agent-vs-workflow-decision.md) — corrective for anti-pattern #1.
- [agent-loop-design.md](./agent-loop-design.md) — corrective for anti-pattern #2 (turn budget) plus related budget infrastructure.
- [agent-cost-control.md](./agent-cost-control.md) — corrective for anti-pattern #3 (cost budget).
- [tool-architecture.md](./tool-architecture.md) — corrective for anti-pattern #4 (idempotency) and #6 indirectly (structured error envelope).
- [memory-engineering.md](./memory-engineering.md) — corrective for anti-pattern #5 (memory taxonomy).
- [error-and-partial-failure.md](./error-and-partial-failure.md) — corrective for anti-pattern #6 (retry discipline).
- [agent-observability.md](./agent-observability.md) — corrective for anti-pattern #7 (trajectory observability).
- [multi-agent-coordination.md](./multi-agent-coordination.md) — related: "multi-agent for everything" is a variant of #1.
- [agent-evals.md](./agent-evals.md) — eval coverage for anti-patterns; trajectory eval surfaces #2, #6 in production.
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — span shape that supports #7's corrective.
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alerts that detect #2, #6, #7-related symptoms.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — gate that blocks regressions toward anti-patterns.
- [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — corrective for #3 at the gateway layer.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — attribution that makes #3 incidents triageable.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/tool-call-authorization.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/tool-call-authorization.md) — architectural side of #4's corrective.
- Sibling repo: ai-security-reference-architecture `agent-security/` folder — security anti-patterns that complement these engineering anti-patterns.
- "An anti-pattern catalogue is a teaching tool" — the practice of consolidated anti-pattern documents is broadly useful; this folder's per-doc anti-pattern sections plus this consolidated catalogue work together.
