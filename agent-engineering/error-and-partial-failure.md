# Error and Partial Failure

> **Audience.** Engineers who own the failure surface of an agent — runner authors, tool authors, the on-call team responding to agent-related incidents. Tech leads designing the failure model for a new agentic feature. **Scope.** Engineering depth on how the agent handles things going wrong — transient and permanent tool failures, junk outputs, misinterpretation, partial multi-step success, rollback and compensating actions. Not the loop runner depth (see [agent-loop-design.md](./agent-loop-design.md)). Not the gateway-level circuit breaker (see [reliability-engineering/fallback-patterns.md](../reliability-engineering/fallback-patterns.md) and [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Agents fail in more ways than single-call LLM systems. A single LLM call has roughly three failure modes: the model errored, the model output was malformed, the model output was wrong. An agent has dozens: tool failed transiently, tool failed permanently, tool returned junk, tool returned partial result, agent misinterpreted a successful tool result, multi-step plan completed half the steps then encountered an unrecoverable state, side-effecting action committed but the next step failed and now state is inconsistent.

The cost of getting failure handling wrong scales with the agent's blast radius. A single LLM call's wrong answer is one bad response. An agent's failure cascade can: bill the same customer twice, send three duplicate emails, leave a multi-step migration half-done, blow a cost budget repairing what didn't need to be repaired, or quietly succeed-on-paper while leaving the underlying system in a broken state.

Most agent failure handling is implicit — written ad-hoc into the runner's loop, or sprinkled across tool implementations, or left to the LLM's discretion ("the model will figure it out"). Implicit handling does not survive contact with production. The taxonomy and corrective patterns belong in code and prompt, not in tribal knowledge.

This document is opinionated about four things:

1. **The failure taxonomy is explicit.** Each category — transient tool failure, permanent tool failure, junk output, misinterpretation, partial multi-step, side-effect rollback — has a different corrective. The agent's runner code and the agent's prompt both have specific handling for each category.
2. **The LLM is not a sufficient error handler on its own.** The runner is the primary error handler; the LLM is one piece. "We'll prompt the model to handle errors" is not a strategy.
3. **Side effects need rollback or compensation discipline.** An agent that takes real-world actions must have a defined behaviour when later steps fail. Either the actions are reversible (rollback), or compensating actions are defined (compensate), or the action is gated by approval (avoid commitment).
4. **Failures are observable and on-call-actionable.** Every category emits a structured signal. The on-call runbook addresses each category specifically.

Structure: (2) the failure taxonomy; (3) transient vs permanent tool failures; (4) junk output handling; (5) misinterpretation of tool results; (6) partial-success handling for multi-step plans; (7) rollback and compensating actions; (8) failure observability and on-call; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The failure taxonomy

Six categories. Each warrants distinct engineering.

### 2.1 Transient tool failure

A tool call failed for a reason that may not recur on retry. Network blip, rate-limit (with backoff), brief unavailability, intermittent timeout. The retry-with-backoff pattern handles these.

### 2.2 Permanent tool failure

A tool call failed for a reason that will recur on retry. Not-found, unauthorized, invalid argument, schema validation failure, tenant-scope violation. Retry is wasteful and can mask the underlying issue. The agent should adapt (try different arguments, try a different tool, or escalate).

### 2.3 Junk output

A tool call "succeeded" — returned a 200 — but the payload is unusable. Empty result where one was expected, garbled text, schema-shaped-but-wrong (e.g., a date in the wrong field), a placeholder ("TODO: implement"). Detect at the boundary and treat as an error envelope; the model handles it as a failure rather than as data.

### 2.4 Model misinterpretation

A tool call succeeded and returned valid data; the agent's next step references the data incorrectly. The patient_id was returned as `"uuid-A"` but the agent passes `"uuid-A "` (with a trailing space) to the next tool. The recent labs list returned 5 entries; the agent's summary mentions 7. The model misread its own input.

### 2.5 Partial multi-step success

A plan with multiple side-effecting steps. Step 1 succeeded; step 2 succeeded; step 3 failed. The system is in a state that's partway between the starting state and the intended end state. Without explicit handling, either the agent declares success (wrong: the intended state is not reached), declares total failure (wrong: steps 1 and 2 still happened), or loops indefinitely trying to make step 3 work (wrong: cost accumulates without progress).

### 2.6 Side-effect inconsistency

A step succeeded but its effect is no longer accurate — a record was created but later steps revealed it shouldn't have been; an email was sent but the agent now needs to retract; an external system was modified but the agent's plan changed. Rollback or compensating actions are required.

### 2.7 The six categories together

| Category | Detection | Primary handler | Agent's response |
| --- | --- | --- | --- |
| Transient | HTTP code / error code marked `retryable: true` | Runner / dispatch layer | Retry with backoff (bounded) |
| Permanent | HTTP code / error code marked `retryable: false` | Runner returns structured error to agent | Adapt (different args/tool) or escalate |
| Junk output | Validator at tool boundary | Tool wrapper rewrites to error envelope | Treat as permanent failure; adapt |
| Misinterpretation | Tool detects (e.g., not-found because args were malformed); eval/observability detects upstream | Runner returns structured error; eval catches systemic issues | Re-derive from prior state |
| Partial multi-step | Workflow / plan controller | Plan controller invokes rollback or compensation | Often non-agent code; agent reports outcome |
| Side-effect inconsistency | Plan controller or compensating-action job | Compensation discipline (idempotent + reversible) | Often outside the agent loop |

Confused systems handle these as one bucket ("error → retry → fail"). Effective systems handle them as six.

---

## 3. Transient vs permanent tool failures

The single most important distinction in the failure taxonomy.

### 3.1 The transient/permanent classification

Every tool error has a classification. Either:

- **`retryable: true`** — transient. Retry with backoff; finite attempts. Examples: rate limit (with `Retry-After`), connection reset, 503 service unavailable, transient timeout.
- **`retryable: false`** — permanent. Do not retry. Examples: 404 not-found, 403 unauthorized, 400 invalid argument, 422 unprocessable, tenant scope violation.

The classification lives in the tool's error envelope (per [tool-architecture.md](./tool-architecture.md) section 5.2):

```json
{
  "status": "error",
  "error_code": "rate_limited",
  "message": "Rate limit hit. Retry after 5 seconds.",
  "retryable": true,
  "retry_after_seconds": 5
}
```

### 3.2 The runner-level retry policy

Transient errors are retried at the *runner / dispatch layer*, not by the agent. The agent doesn't see transient failures that succeed on retry — by the time the runner returns to the agent, the result is either success or an exhausted-retries permanent failure.

Policy:

- **Max retries:** 3 typical. Beyond 3, the tool is treated as permanently failed.
- **Backoff:** exponential with jitter (e.g., 1s, 2s, 4s with ±25% jitter).
- **Honour `retry_after_seconds`** when present.
- **Time-budget aware:** if the agent's time budget is nearly exhausted, skip retries; return the error.
- **Logged at every retry:** spans include retry-count attribute; runner emits a metric.

### 3.3 The agent-level permanent-failure handling

When a tool fails permanently, the runner returns the structured error to the agent in the conversation history. The agent's prompt is engineered to handle structured errors:

```
You may receive tool results with status: "error". When you do:
- Read the error_code, message, and suggested_action.
- If suggested_action indicates trying different arguments, try them.
- If suggested_action indicates trying a different tool, do so.
- If the error indicates the operation is impossible or you are uncertain how to proceed, call escalate_to_care_manager with the error context.
- Do not retry the same call with the same arguments — the runner has already retried transient failures.
```

The prompt + the structured error envelope + the runner's exhausted-retries handling collectively give the agent a coherent error-handling story.

### 3.4 The classification responsibility

Tools authors classify. The classification is part of the tool's contract:

- The tool's source documentation lists error codes and their retryability.
- The tool's tests verify the classification (a forced rate-limit returns retryable=true; a forced unauthorized returns retryable=false).
- Misclassification is a high-priority bug: a permanent failure marked retryable wastes budget; a transient failure marked permanent fails the agent unnecessarily.

### 3.5 Cross-tool classification consistency

The error_code enum (per tool-architecture section 5.2) is consistent across tools. `rate_limited`, `unauthorized`, `not_found`, `invalid_argument`, `tenant_scope_violation`, `tool_internal_error`. The agent's prompt handles the enum, not free-form error strings.

### 3.6 Special case: the LLM itself failing

The LLM call (the model wrapper's call to the provider) is itself subject to transient/permanent classification:

- Transient: 429 rate-limited, 503 unavailable, network timeout.
- Permanent: 400 invalid request (malformed prompt), 401 unauthorized (bad API key).

Handled by the model wrapper (per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md)) with the same retry-with-backoff pattern. The runner sees only the wrapper's final outcome.

---

## 4. Junk output handling

A tool that returns 200 but returns garbage. Detected at the boundary.

### 4.1 The output validator

Every tool's wrapper validates the output before returning it. Validation depends on the tool:

- **Schema validation.** The output matches the declared return schema (types, required fields, formats).
- **Sanity check.** Numeric values within plausible ranges; date fields in plausible ranges; list lengths within configured limits.
- **Content check.** Free-text fields are not empty when they shouldn't be; no telltale placeholder strings ("TODO", "lorem ipsum", "{{ template_var }}").
- **Domain-specific.** Patient IDs match the expected format; clinical codes are recognised values; medication names are in the formulary.

When validation fails, the wrapper returns an error envelope to the runner:

```json
{
  "status": "error",
  "error_code": "tool_invalid_output",
  "message": "Tool returned a result that failed validation: 'date_of_birth' was '0000-00-00'. This typically indicates a missing record or a downstream data issue. Try refining the query or contact support.",
  "retryable": false
}
```

### 4.2 Where validation belongs

The wrapper around the tool, not the agent's prompt. Validating in the prompt ("you may receive junk; check the result before using it") asks the model to do work the validator can do deterministically — wasteful and error-prone.

### 4.3 Validation as eval-surface

The eval surface includes "tool returns junk" cases. The validator must detect known junk patterns; eval verifies. New junk patterns observed in production go to the validator + eval set.

### 4.4 The provenance check

Long-term retrieval is a frequent junk source — the retrieval returns chunks that are too short, all-stopwords, or low-confidence matches. The retrieval tool's validator filters these before returning, or returns them with a "low confidence" flag the agent treats accordingly.

### 4.5 Junk from upstream LLM tools

If a tool's implementation includes an LLM call (e.g., a summariser tool), the inner LLM call can produce junk. The tool's validator catches it (empty summary, summary that's longer than the input, summary that contains the prompt template) and re-runs once before returning an error. Bounded inner repair, not unbounded.

---

## 5. Misinterpretation of tool results

The tool succeeded; the agent's next call references it incorrectly. The pattern is common and corrosive.

### 5.1 The pattern

A typical case:

```
Turn 3:  Agent calls fetch_patient with patient_id="uuid-A".
Turn 3:  Tool returns {patient_id: "uuid-A", name: "Jane Doe", date_of_birth: "1980-01-15", ...}.
Turn 4:  Agent thinks for a moment, then calls fetch_recent_labs with patient_id="uuid-A " (trailing space).
Turn 4:  Tool returns error: patient_not_found.
Turn 5:  Agent thinks "Jane Doe doesn't have recent labs" — wrong inference from a misparsed result.
```

The agent's misinterpretation cascades. The tool returned correct data; the agent's downstream call was wrong; the next tool's error was correctly returned but the agent interpreted it as a feature of the patient rather than a feature of its own argument.

### 5.2 Defensive design at the tool boundary

The tool can detect some misinterpretation:

- **Whitespace tolerance.** `patient_id="uuid-A "` could be treated as `patient_id="uuid-A"` — depending on whether trailing whitespace is meaningful. For most identifiers it isn't; the tool trims and proceeds (logged as a span attribute: `arguments_trimmed: true`).
- **Format normalisation.** Dates accepted in multiple formats; canonicalised before processing.
- **Approximate lookup.** Where reasonable, "did you mean X?" with a structured suggestion field in the response.

These are gentle: they reduce avoidable failures without masking real bugs.

### 5.3 Re-derive-from-state pattern

When the agent encounters a `not_found` on a value it just received, the agent's prompt instructs:

```
If you receive 'not_found' for an identifier you just received from a prior tool call, re-check the identifier value. Common causes: stray whitespace, partial copy, or wrong field referenced. Look at the prior tool result and use the exact identifier value verbatim.
```

The agent re-derives the identifier from the prior state; the next call succeeds.

### 5.4 Structured-extraction state as authority

When the structured state (per [memory-engineering.md](./memory-engineering.md) section 3.3) tracks the current entity, the agent reads from the state rather than from the conversation log. The state is canonical; the conversation log is the diary. Reading from a single canonical source reduces misinterpretation.

### 5.5 Eval surface

Eval cases that test misinterpretation: deliberately ambiguous tool responses, near-duplicate values, identifiers with whitespace. The agent should handle them correctly; failures are tracked.

### 5.6 The "model-can-self-correct" reality check

Frontier models in 2026-Q2 misinterpret less than older models, but the failure mode persists at low single-digit percentages. For high-stakes tools, the cost of misinterpretation is high enough to warrant defensive design. Don't rely on the model's correctness alone.

---

## 6. Partial-success handling for multi-step plans

The agent's plan has N side-effecting steps. K succeeded; step K+1 failed. Now what?

### 6.1 The problem shape

The agent's plan:

1. Create a follow-up appointment proposal.
2. Send notification to patient.
3. Update care plan with the appointment date.
4. Log the action in the EHR.

Steps 1 and 2 succeed. Step 3 fails (transient EHR service issue). The system is in an inconsistent state:

- Proposal exists.
- Patient was notified.
- Care plan is unchanged (still has the old date).
- EHR doesn't have a log entry.

If the agent retries step 3 indefinitely, cost accumulates. If the agent abandons, the patient is notified about an appointment the care plan doesn't reflect. If the agent declares success, the team is misled.

### 6.2 The plan as compensable transaction

The pattern from distributed systems: a multi-step plan is a saga — a sequence of local transactions, each with a defined compensation if the saga fails partway. The agent's plan is engineered the same way.

For each step in a multi-step plan:

- The step's action (what it does on success).
- The step's compensation (what undoes it on saga-failure).
- The step's idempotency (per [tool-architecture.md] section 6).

When step K+1 fails (after retries exhausted), the controller runs compensations for steps K, K-1, ..., 1 in reverse order. The system returns to the starting state.

### 6.3 The plan controller (not the agent)

The plan execution is owned by a controller — workflow code, not the agent's loop. The agent proposes the plan; the controller executes step-by-step with the compensation handling. The agent is informed of the outcome (success, partial-success-rolled-back, partial-success-not-rolled-back-because-some-actions-irreversible).

Why not the agent? Because:

- The agent's loop is unbounded; a controller with strict step-execution is more reliable.
- Compensation logic is deterministic and shouldn't be left to model judgment.
- The controller has the same shape across many features; the agent's plan structure is reused.

The agent's role: produce the plan. The controller's role: execute and handle partial failure.

### 6.4 When compensation isn't possible

Some actions are not reversible. The notification was sent to the patient — you cannot un-send it. The proposal was reviewed by the care manager — you cannot un-review it. The credit card was charged — you cannot un-charge it (you can refund, but the customer-facing effect differs).

For irreversible steps, the discipline shifts:

- **Order matters.** Irreversible steps execute last when possible (after all reversible/idempotent steps succeed).
- **Approval gates.** Irreversible steps require an approval gate (human or rule-based) before commitment.
- **Compensating action instead of rollback.** If you cannot un-do, define a compensating-action (refund, retraction notification, error-correction record).
- **Step-level confirmation.** The plan's design document records "this step is irreversible" so reviewers know.

### 6.5 The plan outcome shape

The controller returns to the agent a structured outcome:

```json
{
  "outcome": "partial_success_rolled_back",
  "completed_steps": ["propose_followup", "send_notification"],
  "failed_step": "update_care_plan",
  "failure_reason": "EHR transient unavailable, retries exhausted",
  "compensations_applied": ["retract_notification", "delete_proposal"],
  "compensations_failed": [],
  "user_facing_message": "I was unable to complete the follow-up scheduling because of a temporary system issue. The proposal and notification have been retracted. Please try again in a few minutes or escalate to your care manager."
}
```

The agent's next turn reads this and informs the user. The trace shows every step.

### 6.6 The "agent shouldn't do multi-step side-effecting plans" alternative

For many agent features, the right answer is to not give the agent multi-step side-effecting plans at all. The agent proposes; a workflow executes; the workflow handles partial failure with its mature primitives (Temporal sagas, Step Functions, etc.). The agent's role narrows to the planning, not the execution.

This is often the right design. The hybrid shape (per [agent-vs-workflow-decision.md](./agent-vs-workflow-decision.md)) puts execution in the workflow shell; the agent is the planner inside one step. See section 9 for the Meridian example.

---

## 7. Rollback and compensating actions

The discipline for agents that take real-world side effects.

### 7.1 The three rollback strategies

**Strategy A — Idempotent + reversible.** The step can be reversed; the reverse operation is itself idempotent. `create_record` → `delete_record`. `update_field` → `revert_field` (using the prior value captured at write). The cleanest case.

**Strategy B — Compensating action.** The step cannot be reversed; a compensating action partially mitigates. `send_email` → `send_correction_email`. `charge_card` → `refund_card`. The user-facing effect is two events instead of zero, but the system is approximately restored.

**Strategy C — Avoid the commitment.** The step does not commit until an approval point. `propose_charge` (no charge yet) + `execute_proposal` (the actual charge). The plan can fail before `execute_proposal` without rollback being needed.

Most production systems use a mix. Strategy C for high-stakes irreversible steps; Strategy A for reversible mutations; Strategy B for inevitably-irreversible communications.

### 7.2 Capturing prior state for revert

Strategy A requires capturing the prior state so the revert knows what to restore. Pattern:

```python
def update_patient_address(patient_id, new_address):
    prior = fetch_patient_address(patient_id)
    apply_update(patient_id, new_address)
    return UpdateResult(
        new_value=new_address,
        prior_value=prior,
        revert_token=create_revert_token(prior),
    )

def revert_update(revert_token):
    state = decode_token(revert_token)
    apply_update(state.patient_id, state.prior_value)
```

The revert_token is returned with the success and can be persisted with the plan's state. Compensation uses it.

### 7.3 The compensating-action design

For Strategy B, the compensating action is a tool of its own — engineered, tested, observable:

- `send_correction_email(prior_email_id, correction_text)` — references the prior send; sends a follow-up that the prior message was incorrect.
- `cancel_appointment(appointment_id, reason)` — cancels and notifies; tracks reason for analytics.
- `refund_charge(charge_id, amount, reason)` — refunds; tracks reason; emits a notification.

The compensating action is itself idempotent (called twice → same outcome). It has its own error envelope.

### 7.4 The approval-gate pattern (Strategy C)

For irreversible-and-high-stakes, commitment is gated by approval (HITL or rule-based). Pattern:

- Agent calls `propose_action(...)` — creates a proposal; no side effect yet.
- Approval system reviews — human or rule-based — and either approves, modifies, or rejects.
- Approval triggers the actual `execute_proposal(proposal_id)` — the side effect occurs only here.
- If the plan fails between propose and execute, the proposal is voided; no rollback needed because no commitment occurred.

The pattern is the right answer for actions where rollback is unacceptable and commit-on-failure is unacceptable.

### 7.5 The compensation log

Every compensation is logged separately from the original action. The audit trail records "step X was completed at T1; compensated at T2 due to reason Y." Without the log, the system's history is opaque.

### 7.6 Testing compensation

Compensation must be tested. Tests:

- **Force a failure mid-plan.** Verify compensations execute in reverse order; verify the system returns to the starting state.
- **Force a compensation failure.** What happens if compensation itself fails? Typically: escalation alert; manual intervention. Verify the alert fires.
- **Test the approval-gate path.** Plan failure between propose and execute; verify proposal is voided.
- **Integration test on a non-prod tenant.** End-to-end with real side effects (real but reversible: e.g., a non-prod email address that captures rather than sends).

### 7.7 The runbook for compensation failure

Compensation can fail (compensation-of-a-compensation is rarely defined). When it does, the on-call runbook covers:

- Alert details (what step failed to compensate; what state is the system in).
- Manual recovery steps (queries to identify affected records; manual fixes).
- Escalation (who to notify if manual recovery isn't straightforward).
- Post-incident: did the compensation fail because of a transient issue (re-run) or a design bug (engineering fix)?

---

## 8. Failure observability and on-call

A failure-handling design without observability is fictitious; you'll never know whether it works.

### 8.1 Per-tool failure metrics

Per tool, per failure category:

- Transient failure rate (count, % of calls).
- Permanent failure rate (count, % of calls, broken down by error_code).
- Validator-detected junk rate.
- Misinterpretation rate (best-effort; some are observable, some are not).
- Average retry count (for retried calls).

### 8.2 Per-feature failure metrics

Per agent feature:

- Plan-level outcomes: success / partial-success-rolled-back / partial-success-not-rolled-back / total-failure.
- Compensation execution rate; compensation failure rate.
- Escalation rate (cases where the agent gave up and called escalate_to_human).
- Cost spent on failed plans vs successful plans.

### 8.3 Alerts

- **Tool error-rate spike.** A tool's error rate jumps; likely an upstream change. On-call paged.
- **Compensation failure.** Any compensation failure is on-call paged; manual recovery typically required.
- **Plan partial-success rate.** Spike indicates a class of failures that's exposing the agent's plan-level handling. Investigation.
- **Escalation rate spike.** Agent is escalating more often; may indicate a regression, a model issue, or a real-world shift in input.
- **Cost on failed plans.** Cost spent on plans that failed exceeds threshold; may indicate retry / repair-loop issue.

### 8.4 The trace surface

Every span carries failure metadata:

- `failure.category` — transient, permanent, junk, misinterpretation, partial, side-effect-inconsistency.
- `failure.error_code`.
- `retry_count`.
- `compensated` (boolean for side-effect-bearing steps).
- `revert_token` (for reversed steps).

The on-call engineer opens an alert, opens the trace, sees the failure cascade, identifies the root cause.

### 8.5 The runbook

The on-call runbook covers each failure category:

- **Tool error-rate spike** → check upstream service status; check the tool's recent deploys; rollback if needed; coordinate with upstream team.
- **Compensation failure** → identify affected records; manual recovery; root-cause analysis; engineering ticket.
- **Plan partial-success spike** → identify the failing step; check its upstream; check for input-shape changes; restore or hotfix.
- **Escalation spike** → eval the recent traces; identify the common failure mode; prompt or tool fix; eval gate.

The runbook is co-located with the agent's deployment; updated when handling changes.

### 8.6 Post-incident review

Every significant failure-related incident triggers a review:

- Was the failure category caught by the existing handler? If not, why?
- Did the observability surface the failure quickly? If not, what to add?
- Did the runbook help? If not, what to update?
- Is the underlying design sound, or is a design change warranted?

Updates land in code, prompts, tools, eval set, runbook.

---

## 9. Worked Meridian example

Meridian's care-coordinator agent's failure handling, in production for ~14 months.

### 9.1 The failure-handling investments

| Investment | Status |
| --- | --- |
| Transient/permanent classification on all tool errors | Standard since launch |
| Bounded retries (3 attempts) with exponential backoff | Standard since launch |
| Output validators on every tool | Phased in over Q1-Q2 |
| Structured-error envelope across all tools | Standard since launch |
| Plan controller for multi-step side-effecting flows | Added in Q3 after incident |
| Compensation discipline for reversible steps | Added in Q3 after incident |
| Approval gates (propose + execute) for irreversible high-stakes | Standard since launch |
| Failure observability (per-tool, per-feature, alerts) | Added in Q2 |
| On-call runbook by category | Added in Q3 |

### 9.2 A representative incident (Q2)

**Symptom.** Cost spike alert for the care-coordinator feature. Cost per request 3x normal.

**Investigation.** Traces showed a particular agent was looping 25–35 turns instead of the typical 4–6. The pattern: a tool was returning malformed dates ("0000-00-00") for patients with incomplete birth records; the agent was attempting to "fix" the date by trying different formats; each attempt failed; the agent kept trying.

**Root causes.**

1. The tool's output validator did not catch the malformed date.
2. The error returned to the agent didn't say "the source data is missing, this is not a format issue" — so the agent kept treating it as a format issue.
3. The agent's prompt didn't have explicit guidance on "if the source data is missing, don't try to repair; escalate."

**Corrective.**

1. Output validator extended to detect placeholder date values.
2. Tool error message rewrote to "patient record is missing date_of_birth; this cannot be fixed at the tool layer."
3. Agent prompt: "When you receive an error indicating source-data is missing or malformed, do not attempt to repair. Either proceed without that field (if optional) or call escalate_to_care_manager with the context."
4. Eval set: cases with malformed source data; verify agent doesn't loop.
5. Budget circuit-breaker: per-tenant cost-cap reduced for the affected agent until the fix landed.

**Outcome.** Loops on malformed-source-data cases dropped to 1–2 turns (the agent escalates immediately). Cost normalized within 48 hours.

### 9.3 A representative plan-level case

**The follow-up scheduling plan.** When a clinician asks the care coordinator to schedule a follow-up:

1. `propose_followup_appointment(...)` — creates proposal (idempotent, has idempotency_key).
2. `send_notification_to_patient(...)` — sends SMS (idempotent, has key).
3. `update_care_plan(...)` — updates the care plan record (idempotent, has key).
4. `log_action_in_ehr(...)` — logs the action (idempotent, has key).

Each step is idempotent. The plan controller (workflow code, not the agent) executes them. If step 3 fails after retries:

- Compensations: void the proposal (step 1's compensation), send a "we couldn't complete this; the prior message was premature" SMS (step 2's compensation).
- Outcome to agent: `partial_success_rolled_back`.
- Agent's response to clinician: "I was unable to complete the scheduling because of a temporary issue with the care plan system. The proposal and notification have been retracted; please try again in a few minutes."

The compensation for step 2 (the notification) is itself a side effect — the patient receives a second SMS — but it's the right behaviour because the first SMS already created an expectation that needs correcting.

### 9.4 A representative misinterpretation case

The patient_id-with-trailing-whitespace pattern (per section 5.1) was observed early. Corrective:

1. Patient-ID tools trim whitespace on input; logged with `arguments_trimmed: true` attribute.
2. Agent prompt: "When you receive an error for an identifier you just received, re-check the value. Common cause: stray whitespace from copy."
3. Eval cases: deliberately ambiguous responses; verify agent handles.

The pattern reoccurred only twice in the 14 months following the fix, both edge cases that didn't warrant further engineering.

### 9.5 The runbook structure

```
== Care-Coordinator Agent Runbook ==
Section 1: Cost spike
Section 2: Error-rate spike (tool)
Section 3: Error-rate spike (LLM)
Section 4: Plan partial-success spike
Section 5: Compensation failure
Section 6: Escalation rate spike
Section 7: Multi-tenant isolation alarm
Section 8: Recent incidents and fixes
```

Each section: symptoms, triage queries (LogQL / SQL), likely causes, immediate mitigation, root-cause investigation pointers, post-incident actions.

### 9.6 What the design avoided

The team explicitly resisted three temptations:

1. **"Let the model handle errors."** The model is not a sufficient error handler; the runner is the primary handler.
2. **"Retry on everything."** The retry policy is bounded and aware of the budgets.
3. **"Big-bang rewrite on incidents."** Each incident produces a targeted fix; the architecture is stable.

---

## 10. Anti-patterns

### 10.1 "Retry forever"

The runner retries any failure indefinitely. Cost accumulates; the agent never returns to a state where it can take a different action.

**Corrective.** Bounded retry policy. Max attempts. Time-budget awareness.

### 10.2 "Errors are strings"

Tool errors are free-form strings. The agent's prompt tries to parse them. Inconsistencies cascade.

**Corrective.** Structured error envelope per [tool-architecture.md](./tool-architecture.md) section 5.2. Error code enum. Retryable boolean.

### 10.3 "No output validator"

Tools return whatever they return. Junk outputs reach the agent. The agent acts on garbage.

**Corrective.** Output validator at every tool boundary. Junk → error envelope.

### 10.4 "Agent does the multi-step plan"

The agent loops, executing side-effecting steps one at a time, with no controller. Partial-failure handling is ad-hoc. Compensation doesn't happen.

**Corrective.** Plan controller (workflow code) owns multi-step execution. Agent proposes; controller executes.

### 10.5 "No compensation for reversible steps"

Side-effecting steps don't have a defined compensation; partial-success cases leave the system inconsistent.

**Corrective.** Compensation per step per section 7. Captured prior state; revert tools; tested.

### 10.6 "Irreversible-and-direct"

The agent has a direct tool for an irreversible action (charge card, send broadcast). No approval gate. Mistakes commit.

**Corrective.** Approval gate per section 7.4 (propose + execute pattern).

### 10.7 "Failure isn't observable"

The trace doesn't show failure categories; metrics don't break out by category; alerts are absent.

**Corrective.** Failure metadata in every span. Per-tool and per-feature metrics. Alerts per section 8.3.

### 10.8 "Runbook is tribal knowledge"

The on-call engineer learns failure handling from prior incidents and chat threads. No documented runbook.

**Corrective.** Runbook per agent feature; sections per failure category; updated on every significant incident.

---

## 11. Findings (sprint-assignable)

### AGT-FAIL-001 — Severity: Critical
**Finding.** Tool errors are unstructured; agent prompt parses free-form strings inconsistently.
**Recommendation.** Structured error envelope per [tool-architecture.md](./tool-architecture.md) section 5.2 with error_code, message, retryable, suggested_action.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-FAIL-002 — Severity: Critical
**Finding.** Runner has unbounded retries; cost incidents traceable to retry loops.
**Recommendation.** Bounded retry policy per section 3.2; honour time budget; alert on retry-exhaustion rate.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-FAIL-003 — Severity: Critical
**Finding.** Agent has direct irreversible-action tool (e.g., charge, broadcast, delete) with no approval gate.
**Recommendation.** Approval-gate pattern per section 7.4; propose + execute.
**Owner.** ai-platform-eng + product, sprint N+1.

### AGT-FAIL-004 — Severity: High
**Finding.** Agent executes multi-step side-effecting plans inside the loop; partial-success cases leave inconsistent state.
**Recommendation.** Plan controller per section 6.3; agent proposes, controller executes with compensation discipline.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-FAIL-005 — Severity: High
**Finding.** Reversible-side-effect tools have no compensation defined; partial failures are not recovered.
**Recommendation.** Per-step compensation per section 7; tested with synthetic failure injection.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-FAIL-006 — Severity: High
**Finding.** Output validators absent on tools that interact with messy upstream data; junk reaches the agent.
**Recommendation.** Validator at each tool boundary per section 4.1; junk → error envelope.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-FAIL-007 — Severity: High
**Finding.** Permanent failures are misclassified as retryable (or vice versa); wasted retries or premature failure.
**Recommendation.** Classification audit; tests per tool per section 3.4; misclassification treated as a high-priority bug.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-FAIL-008 — Severity: High
**Finding.** Compensation failures are silent; no alert when a compensation itself fails.
**Recommendation.** Alert on every compensation failure per section 8.3; runbook section per section 7.7.
**Owner.** ai-platform-eng + ops, sprint N+3.

### AGT-FAIL-009 — Severity: Medium
**Finding.** Agent prompt does not include guidance for handling structured errors; agent retries the same call.
**Recommendation.** Prompt guidance per section 3.3; eval cases for structured-error handling.
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### AGT-FAIL-010 — Severity: Medium
**Finding.** Failure categories are not tagged on spans; on-call cannot triage from the trace.
**Recommendation.** `failure.category` span attribute per section 8.4; per-category metrics.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-FAIL-011 — Severity: Medium
**Finding.** No tests for compensation behaviour; never been verified end-to-end.
**Recommendation.** Compensation test suite per section 7.6; integration tests on non-prod tenant with synthetic failure.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-FAIL-012 — Severity: Medium
**Finding.** Eval set does not cover misinterpretation patterns (whitespace, near-duplicates, ambiguous returns).
**Recommendation.** Misinterpretation cases in eval per section 5.5; track agent's accuracy.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-FAIL-013 — Severity: Medium
**Finding.** Runbook does not have per-category sections; on-call response is improvised.
**Recommendation.** Runbook structure per section 8.5/9.5; sections for each failure category.
**Owner.** ops + ai-platform-eng, sprint N+4.

### AGT-FAIL-014 — Severity: Medium
**Finding.** Idempotency keys not enforced on side-effecting tools; retries can double-execute.
**Recommendation.** Idempotency per [tool-architecture.md] section 6.2; runtime de-duplicates.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-FAIL-015 — Severity: Low
**Finding.** Tool whitespace tolerance (per section 5.2) is inconsistent across tools.
**Recommendation.** Consistent tolerance policy; logged when normalization happens.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-FAIL-016 — Severity: Low
**Finding.** Post-incident reviews are not always tied to handler updates; the same incident pattern recurs.
**Recommendation.** Post-incident review template per section 8.6; review-to-update traceability.
**Owner.** ai-platform-eng + ops, sprint N+5.

### AGT-FAIL-017 — Severity: Low
**Finding.** Compensation log is not separately queryable; audit "what compensations ran" is difficult.
**Recommendation.** Compensation log per section 7.5 as a queryable surface; dashboard panel.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-FAIL-018 — Severity: Low
**Finding.** Failure-related metrics live in different places; no consolidated view.
**Recommendation.** Failure dashboard per agent feature; sections per section 8.1/8.2.
**Owner.** ai-platform-eng + ops, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team building failure handling from scratch:

- [ ] **Sprint 0 — taxonomy adoption.** Adopt the six-category taxonomy; document per-tool classifications.
- [ ] **Sprint 1 — structured error envelope.** Standard across all tools.
- [ ] **Sprint 1 — bounded retry policy.** Runner-level transient retry.
- [ ] **Sprint 1 — output validators.** At every tool boundary.
- [ ] **Sprint 2 — prompt for structured errors.** Agent handles errors deliberately.
- [ ] **Sprint 2 — approval gate for irreversibles.** Propose + execute pattern.
- [ ] **Sprint 2 — failure observability.** Span attributes, metrics, alerts.
- [ ] **Sprint 3 — plan controller.** If multi-step side-effecting plans exist.
- [ ] **Sprint 3 — compensation per step.** Reversible steps have compensation; tested.
- [ ] **Sprint 3 — runbook.** Per-category sections.
- [ ] **Sprint 4 — eval cases.** Per-category failure cases; verify handling.
- [ ] **Sprint 4 — compensation testing.** End-to-end on non-prod tenant.
- [ ] **Ongoing — post-incident reviews.** Update code, prompts, tools, eval, runbook.

For a team retrofitting existing failure handling:

- [ ] **Sprint 0 — audit.** What categories does the system handle? Where are the gaps? Which incidents in the last quarter map to which categories?
- [ ] **Sprint 1 — fix the worst category.** Often the lack of plan controller or lack of approval gates.
- [ ] **Sprint 1 — observability.** Make failures visible.
- [ ] **Sprint 2 — fix the next worst.** Per the audit.
- [ ] **Sprint 3 — runbook.** Document what's been built.
- [ ] **Sprint 3 — eval coverage.** Add cases for the categories that had gaps.

A team that completes the sequence has an agent whose failure model is engineered, observable, and recoverable. A team that doesn't has an agent whose incidents are surprises and whose recovery is improvised.

---

## 13. References

- [agent-engineering-playbook.md](./agent-engineering-playbook.md) — broader practice; section 6 (error handling).
- [agent-loop-design.md](./agent-loop-design.md) — runner where retry, termination, and error handling live.
- [tool-architecture.md](./tool-architecture.md) — error envelope, idempotency, side-effect categories.
- [memory-engineering.md](./memory-engineering.md) — memory failures share the same disciplines.
- [agent-observability.md](./agent-observability.md) — trajectory observability that surfaces failures.
- [agent-cost-control.md](./agent-cost-control.md) — cost-control patterns interact with retry policy and partial-failure handling.
- [agent-evals.md](./agent-evals.md) — eval coverage for failure cases.
- [reliability-engineering/fallback-patterns.md](../reliability-engineering/fallback-patterns.md) — gateway-level fallback patterns that complement per-tool retry.
- [reliability-engineering/retry-strategy.md](../reliability-engineering/retry-strategy.md) — broader retry strategy patterns (planned doc).
- [reliability-engineering/circuit-breakers.md](../reliability-engineering/circuit-breakers.md) — circuit breaker patterns at the platform layer (planned doc).
- [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — cost-based breaker that prevents runaway-cost incidents.
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — span shape that carries failure metadata.
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alerting patterns for the failure categories.
- Sibling repo: [ai-architecture-reference-architecture/integration-architecture/integration-failure-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture) — architectural integration-failure patterns (planned).
- Sagas (Garcia-Molina & Salem, 1987; Microservices Patterns, Richardson, 2018) — distributed-systems compensation discipline that informs section 6.
- Temporal, AWS Step Functions, Inngest — workflow engines whose saga patterns are the reference implementations for compensation discipline.
