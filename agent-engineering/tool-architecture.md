# Tool Architecture

> **Audience.** Engineers designing the tool surface an agent calls — tool catalog owners, agent runner authors, tech leads making the MCP vs local-function call. **Scope.** Tool design as a first-class engineering concern: naming and descriptions, argument schemas, return shapes, idempotency and side-effect boundaries, the registry pattern, MCP-vs-local decision, and testing tools with the agent. Not the loop runner (see [agent-loop-design.md](./agent-loop-design.md)). Not tool authorization policy (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `guardrails-and-policy-architecture/tool-call-authorization.md`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The tool surface is the single most leveraged piece of agent engineering. A poorly designed tool surface produces a confused agent regardless of model quality — wrong tools selected, wrong arguments provided, results misinterpreted, infinite repair loops trying to recover from a tool's bad error message. A well-designed tool surface lets a smaller and cheaper model perform far above its weight: the model picks the right tool the first time, supplies the right arguments, gets a useful result, and proceeds.

The asymmetry is large. Teams routinely spend weeks tuning prompts and selecting bigger models when the actual failure mode is a tool whose description is ambiguous, a tool whose error message is unparseable, or a tool registry that exposes 47 tools when 12 would suffice. Refactoring the tool surface is often the single highest-leverage change a team can make to an agent's quality and cost.

This document is opinionated about four things:

1. **Tools are designed for the model.** A tool is not a wrapper around an API for engineers' convenience. It is a model-facing interface. The naming, description, argument shape, and return shape are all written with one question in mind: will a model that hasn't seen this before pick the right tool, supply the right arguments, and understand the result?
2. **The tool's *description* is its most important attribute.** The model picks tools by reading their descriptions. A description that's right for engineers ("internal helper for patient_v3") is wrong for the model ("fetches a patient's demographics and recent visits given a patient_id").
3. **The catalogue must be curated.** Every tool added widens the model's decision space. A 50-tool catalogue produces worse model decisions than a 10-tool catalogue even when the 50 tools cover more capability — the model spends attention disambiguating between similar tools instead of getting the answer right. Removing tools is a positive engineering action.
4. **Idempotency and side-effect boundaries are not optional.** Agents loop. Tool calls retry. A tool that side-effects without idempotency will produce duplicates in production. The discipline applies to every side-effecting tool from day one.

Structure: (2) the tool surface as engineering object; (3) tool naming and description; (4) argument schemas; (5) return shapes — success, error, partial; (6) idempotency and side-effect boundaries; (7) the tool registry pattern; (8) MCP vs local function decision; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The tool surface as engineering object

A tool is a model-callable function. Its model-visible contract has four parts: name, description, argument schema, and return shape. Its runtime behaviour has three parts: dispatch, side effect (if any), and observability.

### 2.1 The four model-visible attributes

- **Name.** Short, lowercase-snake-case, action-oriented. The model uses the name to disambiguate among tools when descriptions are similar.
- **Description.** A few sentences. States what the tool does, when to call it, what it returns, and what *not* to use it for. The most important attribute for selection accuracy.
- **Argument schema.** A JSON Schema (or framework equivalent). Names, types, descriptions per argument. The argument descriptions matter as much as the tool description.
- **Return shape.** Either a success payload (typed) or an error envelope (structured). The model sees the JSON; structure it for model consumption, not just engineer convenience.

### 2.2 The three runtime concerns

- **Dispatch.** The agent runner receives a tool call and dispatches to the tool's implementation. The dispatch is policed by the registry (next section) for tenant scope, authorization, and budgets.
- **Side effect.** If the tool mutates state outside the agent (sends an email, creates a record, charges a card, modifies a file), the side effect is governed by idempotency and a side-effect contract (section 6).
- **Observability.** Each tool call emits a span with arguments, latency, cost (if any), success/error, and the return payload. Per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) and [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md).

### 2.3 What a tool is not

- **A thin pass-through to an API.** A tool may wrap an API but the wrapping is shaped for the model, not the API. The API might return `{ "p_id": "123", "fn": "Jane", "ln": "Doe", "dob_us": "01/15/1980" }`; the tool returns `{ "patient_id": "123", "first_name": "Jane", "last_name": "Doe", "date_of_birth": "1980-01-15" }`. The tool is the translator.
- **Every internal function exposed.** A tool catalogue is the curated subset of internal capabilities you want the agent to use. Most internal functions are not tools.
- **A way to give the agent generic capability.** "Bash command tool" or "any-HTTP tool" — these expose unbounded capability and unbounded liability. Specific tools for specific tasks.
- **A workflow step.** A workflow step is engineer-callable; a tool is model-callable. They have different design constraints.

### 2.4 The "tool" vs "function" distinction in 2026

Provider SDKs use "function" and "tool" somewhat interchangeably (Anthropic's tool use, OpenAI's function calling, etc.). This document uses *tool* throughout because it captures the model-facing interface concept regardless of how the provider names it.

---

## 3. Tool naming and description

The naming and description are the model's selection surface. Get them right.

### 3.1 Naming conventions

- **Verb-noun, lowercase, underscores.** `fetch_patient`, `search_documents`, `send_message`, `escalate_to_human`.
- **Verbs that convey action shape.** `fetch_` for read of a known thing. `search_` for query of an unknown thing. `list_` for enumeration. `get_` is too generic. `create_`, `update_`, `delete_` for mutations. `send_`, `submit_`, `notify_` for outbound side effects.
- **Nouns that are specific.** `patient_demographics`, not `patient` (which patient field? the model has to guess). `recent_lab_results`, not `labs` (recent? all? specific test?). Specificity reduces the model's disambiguation burden.
- **Avoid acronyms unless universal.** `fetch_patient_ehr` is fine if EHR is universal in the domain. `fetch_pt_dm_v3` is opaque.
- **Avoid version suffixes.** `fetch_patient_v2` invites the model to ask why there's a v1, or to call the wrong version. Version internally; the model sees one name.

### 3.2 Description anatomy

A good description has four parts:

1. **What it does.** One sentence. "Fetches a patient's demographics and contact information."
2. **When to call it.** "Call this when you need name, date of birth, address, phone, email, or insurance card on file."
3. **What it returns.** "Returns a `PatientDemographics` object with `patient_id`, `name`, `date_of_birth`, `address`, `phone`, `email`, `insurance`. Returns an error envelope if the patient_id is not found or the tenant lacks access."
4. **What *not* to use it for.** "Do not use this for clinical data, encounters, or lab results — use `fetch_patient_encounters`, `fetch_recent_labs`, or `search_clinical_notes` for those."

The negative space ("what not to use it for") is what most tool descriptions lack. The model selects based on positive description alone, which produces a near-tie between similar tools. Explicit exclusions break the tie.

### 3.3 Description as model-consumed text

The model reads every tool's description on every turn. The descriptions are tokens that count toward context. Discipline:

- **A few sentences per tool, not paragraphs.** Long descriptions waste context and obscure the signal.
- **Concrete, not abstract.** "Fetches the patient's record" is abstract; "fetches name, DOB, address, phone, email, and insurance" is concrete.
- **Use the model's vocabulary.** The model is trained on common terms (`patient`, `appointment`, `medication`); use those, not internal jargon (`encounter_entity`, `med_event`).
- **State error behaviour.** "Returns an error envelope if the patient_id is not found" — the model knows what to expect and can recover.

### 3.4 Disambiguation between similar tools

When you have similar tools — `search_patients` and `fetch_patient` — make the disambiguation explicit:

- `search_patients`: "Search for patients matching criteria (e.g., name, DOB). Returns a list of candidates with their `patient_id`s. Use this when you do not have a specific patient_id and need to find one."
- `fetch_patient`: "Fetch a specific patient's record by `patient_id`. Use this only when you already have a confirmed `patient_id` (typically from a prior `search_patients` call)."

The pair makes the workflow legible: search first, then fetch by id. Without disambiguation the model might call `fetch_patient` with `patient_id: "Jane Doe"` and get an error.

### 3.5 Description rot

Descriptions decay as the implementation evolves. The discipline:

- **Description is part of the tool's contract.** PR review covers the description as much as the code.
- **Eval set covers the description.** If the description says "returns an error envelope when patient_id is not found," the eval set includes a case where the model is given a bad patient_id and the model is expected to handle the error gracefully. The eval forces description accuracy.
- **Periodic description audit.** Quarterly review of the catalogue; reread each description against the implementation; update or delete stale tools.

---

## 4. Argument schemas

The argument schema is what the model fills in when calling the tool. Errors here are common and expensive.

### 4.1 Schema structure

Use JSON Schema (or framework equivalent). Every argument has a name, type, description, and (where applicable) constraints.

```json
{
  "name": "search_patients",
  "description": "Search for patients matching criteria. Returns up to 25 candidates.",
  "input_schema": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string",
        "description": "Full or partial name. Case-insensitive substring match."
      },
      "date_of_birth": {
        "type": "string",
        "format": "date",
        "description": "Patient's DOB in ISO format YYYY-MM-DD."
      },
      "phone": {
        "type": "string",
        "description": "Phone number, digits only, 10 digits. Optional."
      }
    },
    "required": ["name"]
  }
}
```

### 4.2 Per-argument descriptions

Each argument's description is a directive to the model. Tell the model:

- **The format.** "ISO date YYYY-MM-DD." "Phone digits only, 10 digits." "Patient ID is a UUID."
- **The meaning.** Not "name" but "patient's full or partial name."
- **Constraints.** "Maximum 100 characters." "Must be one of: appointment, lab, encounter."
- **When it's optional.** "Optional. If omitted, all visit types are returned."

Argument-description quality is as load-bearing as tool-description quality. The model fills in the wrong format because the description was silent on format.

### 4.3 Strict schemas

Use strict schemas:

- **Required fields are required.** The model is forced to supply them; the runtime rejects calls that omit them with a descriptive error.
- **Types are checked.** `date_of_birth: "01/15/1980"` fails schema validation (wrong format); the runtime returns an error the model can correct.
- **Enums where applicable.** `visit_type: enum["appointment", "lab", "encounter"]` constrains the model's choices; reduces hallucinated values.
- **Constraints are enforced.** `string` with `format: date` is enforced; out-of-range values are rejected.

The error from schema validation must be model-readable (section 5) so the model can correct and retry.

### 4.4 Avoiding common argument schema mistakes

- **Don't accept unbounded text.** A tool whose argument is "search query: any string" is unbounded; the model may pass paragraphs. Constrain to what's useful.
- **Don't accept code as a string.** "Run this Python: ..." is dangerous and produces bad model behaviour. Specific tools for specific actions.
- **Don't accept "options" as a JSON blob.** The model invents fields. Specific named arguments.
- **Don't accept date ranges as strings.** "last week", "this month" — the model picks values; ambiguity. Pass `start_date` and `end_date` as ISO dates.
- **Don't omit a `limit` on listy tools.** `search_patients` without a limit may return thousands; the model is overwhelmed and burns context. Default and maximum limit.

### 4.5 Schema versioning

Schemas evolve. A schema change can break in-flight conversations (if the model proposes a call with the old schema and the runtime rejects it under the new schema). Discipline:

- **Schema changes are backward-compatible where possible.** Adding optional arguments is safe. Renaming or removing arguments is breaking.
- **Breaking schema changes have a deprecation window.** Old schema and new schema coexist; tool description points to the new shape; old shape is removed after the window.
- **Versioned tool names for major changes.** If the semantics genuinely change, ship a new tool (`fetch_patient_v2`) and deprecate the old one rather than mutating the old one's schema.

---

## 5. Return shapes — success, error, partial

What the tool returns is what the model reads back. Structure it.

### 5.1 Success payload

A typed payload the model can interpret. Engineer the field names and structure for model legibility:

```json
{
  "status": "success",
  "patient": {
    "patient_id": "uuid-xxxx",
    "first_name": "Jane",
    "last_name": "Doe",
    "date_of_birth": "1980-01-15",
    "address": "123 Main St, Springfield",
    "phone": "5551234567",
    "email": "jane.doe@example.com",
    "insurance": {
      "carrier": "Aetna",
      "plan": "PPO 2000",
      "member_id": "AT-12345"
    }
  }
}
```

The model reads this as JSON; field names guide interpretation. Avoid:

- **Cryptic field names.** `pt_dob` confuses; `date_of_birth` is clear.
- **Nested arrays of unlabeled values.** `[1980, 1, 15]` requires the model to guess; `{year: 1980, month: 1, day: 15}` is explicit.
- **Internal IDs that confuse.** If the model needs the `patient_id`, return it as `patient_id`, not `record_pk`.
- **Massive payloads.** A patient record might have 200 fields. Return the 12 the model needs; provide a `fetch_full_patient_record` tool if the model needs more.

### 5.2 Error envelope

Errors are the model's chance to correct. Make the envelope structured and informative:

```json
{
  "status": "error",
  "error_code": "patient_not_found",
  "message": "No patient found with patient_id 'xyz'. The patient_id should be a UUID. If you have a name or other identifier, try search_patients first.",
  "retryable": false
}
```

Fields:

- **`status: "error"`** — distinguishes from success.
- **`error_code`** — machine-readable enum (`patient_not_found`, `unauthorized`, `rate_limited`, `invalid_argument`, `tool_internal_error`, `tenant_scope_violation`).
- **`message`** — human-readable, model-readable. Says what went wrong, why, and how to recover.
- **`retryable`** — boolean. `true` for transient errors (rate-limited, service unavailable); `false` for permanent (not found, unauthorized, invalid argument).
- **Optional `suggested_action`** — a hint to the model: "Try search_patients with the patient's name."

A good error tells the model what to do next. A bad error ("Internal error") sends the model into a repair loop.

### 5.3 Partial results

Some tools succeed partially — found 10 results out of an unknown total, retrieved 8 fields out of 12 requested, processed 50 records before timing out. The envelope:

```json
{
  "status": "partial",
  "results": [...],
  "partial_reason": "result_limit_reached",
  "message": "Returned the first 25 matches. There may be more results; refine your search criteria or specify additional filters."
}
```

The model knows the result is partial and can decide whether to refine or continue.

### 5.4 Payload size discipline

Return payloads count toward context. Discipline:

- **Limit list sizes.** Default and maximum limits on list-returning tools.
- **Summarise large fields.** A 50-page document returned as full text is wasteful. Return a summary + offsets; provide a `fetch_document_section` tool for drill-in.
- **Truncate with explicit marker.** If a field is truncated, indicate it: `"description": "First 500 chars of description...[truncated; total length 12,400]"`.
- **Drop fields the model doesn't need.** A patient record may have internal fields (audit trail, source system metadata); these belong in the engineering surface, not the model surface.

### 5.5 Consistency across tools

The success/error/partial envelope is consistent across all tools in the catalogue. The model learns the envelope shape once and applies it across the catalogue. Inconsistency forces the model to learn each tool's quirks and produces more parsing errors.

---

## 6. Idempotency and side-effect boundaries

Agents loop. Tool calls retry. Side effects without idempotency double-execute. This section's discipline applies to every side-effecting tool.

### 6.1 The four side-effect categories

- **Read-only.** No side effect. Idempotency is automatic. Most tools.
- **Idempotent mutation.** Same arguments, same outcome regardless of count. `update_patient_email(patient_id, email)` — calling twice with the same email leaves the same state. Idempotency by design.
- **Non-idempotent mutation with key.** Mutation that is idempotent given an idempotency key. `send_appointment_reminder(patient_id, appointment_id, idempotency_key)` — duplicate calls with the same key are no-ops; only the first sends the SMS. Idempotency via explicit key.
- **Non-idempotent side effect.** `charge_credit_card(amount, card_token)` — calling twice charges twice. Requires explicit handling.

### 6.2 The idempotency-key pattern

For tools in category 3, the model is required to supply an idempotency key. The runtime de-duplicates:

```json
{
  "name": "send_message",
  "description": "Send a message to a patient. The idempotency_key MUST be unique per intended send; do not reuse a key for distinct messages. If the same key has been used recently, this tool is a no-op and returns the prior result.",
  "input_schema": {
    "type": "object",
    "properties": {
      "patient_id": {"type": "string"},
      "message_body": {"type": "string", "maxLength": 1000},
      "idempotency_key": {
        "type": "string",
        "description": "A UUID you generate for this specific send. The runtime de-duplicates retries with the same key. Use a fresh UUID for each distinct message."
      }
    },
    "required": ["patient_id", "message_body", "idempotency_key"]
  }
}
```

The runtime stores the key + result for a TTL (typically 24h); re-calls with the same key return the cached result. The model can retry safely.

### 6.3 The "no side-effect without explicit approval" pattern

For tools in category 4 (non-idempotent, high-stakes), the agent does not directly execute the side effect. The agent proposes; a deterministic process (HITL approval or a verification step) executes.

```
agent: calls propose_charge(amount=500, card_token=xyz, reason="copay")
       returns proposal_id=abc
       no side effect occurred

approval system: enqueues proposal abc for human approval (or rule-based auto-approve)
                 once approved, charges the card
                 emits notification

agent: gets notified of approval; proceeds
```

The agent's tool surface includes `propose_*` tools (no side effect, idempotent: same proposal arguments produce the same proposal ID). The execution layer is outside the agent's loop.

See sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `guardrails-and-policy-architecture/tool-call-authorization.md` for the architectural side of this pattern.

### 6.4 The "verify before mutate" pattern

For mutations where the agent's state may be stale, the tool itself verifies:

```python
def update_patient_address(patient_id, new_address, expected_current_address):
    current = fetch_patient_address(patient_id)
    if current != expected_current_address:
        return error_envelope(
            code="state_changed",
            message=f"Patient's current address is '{current}', not the '{expected_current_address}' you specified. Re-fetch the patient and retry."
        )
    apply_update(patient_id, new_address)
    return success_envelope(...)
```

The agent provides `expected_current_address` (which it observed earlier). The tool checks before mutating. Concurrent edits are detected and surfaced; the model re-fetches and retries.

### 6.5 The "agent has no destructive primitives" rule

The model should not have tools that delete data, drop tables, terminate accounts, or take other irreversible actions without explicit human approval. The blast radius is too large. The pattern is `propose_deletion` + human approval + scheduled execution, not `delete_now`.

### 6.6 The idempotency observability

Every tool call is logged with its idempotency key (if applicable) and whether the call was a fresh execution or a de-duplicated retry. The trace shows which tool calls were no-ops; debugging is straightforward when "the agent sent the same message twice" turns out to be "the agent called send_message twice but the second was de-duplicated."

---

## 7. The tool registry pattern

The registry is the dispatch and policy surface. It is platform code; tools are registered with it.

### 7.1 Responsibilities

- **Catalogue.** Holds the tools available to a given agent.
- **Per-agent scoping.** Different agents see different subsets of the catalogue. The care-coordinator agent sees a clinical tool set; the patient-API copilot sees a developer-tool set.
- **Per-tenant scoping.** Tenant-specific tools (e.g., a tool that queries only Tenant A's data) are scoped at registration.
- **Authorization.** Each tool dispatch checks the call against the tenant scope, the user's authorization, and any tool-specific policy. Unauthorized calls fail with a structured error.
- **Argument validation.** The registry validates the model's arguments against the tool's schema before dispatch. Invalid arguments produce a model-readable error.
- **Dispatch.** Calls the tool's implementation with validated arguments and the call context (tenant, user, trace).
- **Observability.** Emits the tool-call span per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md).
- **Cost.** If the tool incurs cost (LLM-as-tool, external API with metered cost), records the cost into the call context.
- **Rate limiting.** Per-tool, per-tenant, per-user limits where applicable.

### 7.2 Registry shape

```python
class ToolRegistry:
    def register(self, tool: Tool, scopes: list[Scope]) -> None: ...
    def list_for_agent(self, agent_name: str, context: CallContext) -> list[ToolSpec]: ...
    def dispatch(self, tool_name: str, arguments: dict, context: CallContext) -> ToolResult: ...
```

`list_for_agent` returns the subset visible to the agent given the call context. `dispatch` validates, authorizes, executes, and returns the result.

### 7.3 The catalogue curation discipline

- **Each tool's addition is a PR with eval impact analysis.** Adding a tool changes the agent's behaviour; the eval set must include cases that exercise the new tool, and the existing eval results are re-run to detect regression.
- **Periodic removal of unused tools.** The trace data shows which tools are called and which aren't. Tools the agent never picks (because the description is bad, the function overlaps another tool, or the capability isn't needed) are deleted.
- **Catalogue size caps.** A typical agent has 5–15 tools. > 25 is a red flag; the model's attention is over-divided. Either split into multiple agents (with different tool subsets) or remove low-value tools.
- **Description audit cadence.** Quarterly. Per section 3.5.

### 7.4 Per-agent tool subsets

A care-coordinator agent's tool subset differs from a patient-summary agent's tool subset. The registry supports defining the subset per agent:

```python
care_coordinator_tools = [
    fetch_patient,
    search_patients,
    fetch_recent_encounters,
    fetch_recent_labs,
    search_clinical_notes,
    propose_followup_appointment,
    escalate_to_care_manager,
]

patient_summary_tools = [
    fetch_patient,
    fetch_recent_encounters,
    fetch_recent_labs,
]
```

Smaller subsets produce better model decisions. Resist the temptation to give every agent every tool.

### 7.5 Tool authorization

Per [ai-architecture-reference-architecture/guardrails-and-policy-architecture/tool-call-authorization.md]. Summary: every tool call is authorized against the call context (tenant, user role, current task). Unauthorized calls fail with a structured error the model handles (typically by trying a different approach or escalating).

### 7.6 Registry observability

- One span per tool call (per agent-step-instrumentation).
- Span attributes: tool name, validated arguments, success / error / partial, latency, cost (if applicable), authorization decision.
- Aggregations: per-tool call rate, per-tool error rate, per-tool latency, per-tool cost.
- Alerts: tool error-rate spike (an upstream change broke the tool); tool cost spike (model is calling a tool more than expected).

---

## 8. MCP vs local function decision

Tools can be implemented as local functions (in-process code) or via Model Context Protocol (MCP) servers (out-of-process, network-callable). The choice has operational and security implications.

### 8.1 Local function

A Python (or framework-language) function in the agent's process. Direct dispatch.

**Pros.**
- Lowest latency (no network hop).
- Simplest deployment (no extra server).
- Type-safe end-to-end (compiler/typechecker enforces argument and return shapes).
- Easy testing (mock the function).

**Cons.**
- Coupled to the agent's runtime — can't share the tool with another agent in a different language without reimplementation.
- Coupled to the agent's deployment — updates require redeploying the agent.
- Security surface is in-process (no sandboxing between agent and tool).

### 8.2 MCP server

A separate process implementing the MCP protocol. The agent's runtime talks to the MCP server over local socket, stdio, or network. The MCP server is potentially in a different language, different process, different host.

**Pros.**
- Language-agnostic — a Python agent can use a Go MCP server.
- Decoupled deployment — the MCP server can be updated independently.
- Reusable — the same MCP server can serve multiple agents.
- Process isolation — the MCP server runs in its own process; bugs and security issues are contained.
- Ecosystem — many third-party MCP servers exist (filesystem, git, slack, etc.) that the team did not have to build.

**Cons.**
- Higher latency (process or network hop).
- More moving parts (MCP server is its own runtime with its own observability requirements).
- Authentication and authorization are at the MCP boundary, not in-process — must be engineered.
- Schema is JSON over the wire, not native types.

### 8.3 The decision

| Criterion | Local function | MCP |
| --- | --- | --- |
| Latency is critical (< 50ms tool overhead) | Yes | No |
| Reused across multiple agents | Either, but MCP simpler | Yes |
| Reused across multiple languages | No | Yes |
| Independent deployment cadence | Maybe (deploy whole agent) | Yes |
| Process isolation needed (sandbox dangerous capabilities) | No | Yes |
| Third-party tool already exists as MCP | Re-implement | Use |
| Team has MCP operational experience | Either | Yes |

Default: **start with local functions** for tools you own and that have low operational complexity. **Use MCP** for tools that need process isolation (sandboxed code execution, untrusted code), are widely reused, or already exist as third-party MCP servers.

A hybrid catalogue is fine: most tools are local functions, a few sensitive ones are MCP-served.

### 8.4 MCP server engineering

If you build MCP servers, they get the same disciplines as the agent's tools: naming, descriptions, schemas, error envelopes, idempotency, observability. The MCP boundary doesn't relax any of the disciplines.

### 8.5 The "third-party MCP server" decision

Third-party MCP servers can be installed easily; the operational risk is significant:

- **Capability scope.** A filesystem MCP server may expose `read_file` and `write_file` over arbitrary paths. Confirm scope.
- **Authentication.** What credentials does the MCP server hold? Where? Rotation? Audit?
- **Output trust.** The MCP server's return is read by the agent; a malicious server could inject prompt-injection payloads. Validate inputs and outputs at the boundary.
- **Update cadence.** Who updates the MCP server, and on what trigger?

See sibling [ai-security-reference-architecture]'s `agent-security/mcp-server-hardening.md` for the security depth.

### 8.6 Testing tools with the agent

A tool is correct in isolation if its function works given its inputs. A tool is correct *with the agent* if the agent picks it for the right cases, supplies the right arguments, and interprets the result correctly.

The eval pattern:

- **Per-tool unit tests.** Standard function tests. Fast.
- **Tool-with-agent integration eval.** Run the agent on a golden set; trace shows which tools were called with which arguments. Verify per-tool: was the tool picked when it should have been? Were the arguments well-formed? Did the agent recover from a structured error?
- **Negative cases.** Give the agent an input that should *not* trigger the tool; verify the tool wasn't called. False positives are as harmful as false negatives.

The integration eval surfaces tool description and schema issues that unit tests miss. See [agent-evals.md](./agent-evals.md).

---

## 9. Worked Meridian example

Meridian's care-coordinator agent has a curated catalogue of 12 tools. Selected examples below.

### 9.1 `fetch_patient`

```python
@tool(
    name="fetch_patient",
    description=(
        "Fetch a patient's demographics and contact information by patient_id. "
        "Returns first_name, last_name, date_of_birth, address, phone, email, "
        "and insurance carrier/plan/member_id. "
        "Use this when you already have a confirmed patient_id from search_patients or from the user's input. "
        "Do not use this for clinical data (use fetch_recent_encounters, fetch_recent_labs, "
        "or search_clinical_notes); do not use this to search for a patient by name (use search_patients)."
    ),
    input_schema={
        "type": "object",
        "properties": {
            "patient_id": {
                "type": "string",
                "format": "uuid",
                "description": "Patient's UUID. Obtain from search_patients or from confirmed user input.",
            }
        },
        "required": ["patient_id"],
    },
    side_effect="read_only",
)
def fetch_patient(patient_id: str, context: CallContext) -> PatientDemographics:
    # ...
```

**Why this works.** The description tells the model when to use, what it returns, and explicitly what *not* to use it for. The schema constrains `patient_id` to UUID format, so the model can't pass "Jane Doe" and confuse itself. The description points to the right tool for each alternative use case.

### 9.2 `propose_followup_appointment`

A side-effecting tool that uses the propose pattern.

```python
@tool(
    name="propose_followup_appointment",
    description=(
        "Propose a follow-up appointment for a patient. This does NOT schedule the appointment; "
        "it creates a proposal that the patient's care manager reviews and approves. "
        "Once approved, the appointment is scheduled and the patient is notified. "
        "Use this when the patient's care plan indicates a follow-up is appropriate. "
        "Provide a clear clinical_rationale (the care manager will read it). "
        "Each proposal needs a fresh idempotency_key (UUID). "
        "Returns proposal_id, status='pending_approval', and the estimated approval time."
    ),
    input_schema={
        "type": "object",
        "properties": {
            "patient_id": {"type": "string", "format": "uuid"},
            "proposed_visit_type": {
                "type": "string",
                "enum": ["primary_care", "specialist_followup", "lab_only", "imaging"],
            },
            "proposed_window_days": {
                "type": "integer",
                "minimum": 7,
                "maximum": 90,
                "description": "Approximate window in days from today.",
            },
            "clinical_rationale": {
                "type": "string",
                "maxLength": 500,
                "description": "Short clinical rationale (max 500 chars) the care manager will read.",
            },
            "idempotency_key": {
                "type": "string",
                "format": "uuid",
                "description": "Fresh UUID for this specific proposal. Reusing keys yields a no-op.",
            },
        },
        "required": [
            "patient_id",
            "proposed_visit_type",
            "proposed_window_days",
            "clinical_rationale",
            "idempotency_key",
        ],
    },
    side_effect="proposal_only",
)
def propose_followup_appointment(...) -> ProposalResult:
    # ...
```

**Why this works.** The model cannot schedule appointments directly (correct safety boundary). The proposal goes to a human care manager. The enum constrains visit type. The required `clinical_rationale` ensures the care manager has context. The idempotency_key requirement keeps retries safe.

### 9.3 `escalate_to_care_manager`

```python
@tool(
    name="escalate_to_care_manager",
    description=(
        "Escalate the current conversation to a human care manager for follow-up. "
        "Use this when: (a) the question requires clinical judgement beyond what you should provide; "
        "(b) the patient has expressed urgency, distress, or safety concern; "
        "(c) you have exhausted available tools and cannot answer authoritatively; "
        "(d) you are uncertain and a wrong answer could harm patient care. "
        "Provide a clear escalation_reason and any context the care manager will need. "
        "Returns an escalation_id and the estimated response time."
    ),
    input_schema={
        "type": "object",
        "properties": {
            "patient_id": {"type": "string", "format": "uuid"},
            "escalation_reason": {
                "type": "string",
                "enum": [
                    "clinical_judgement_required",
                    "patient_urgency",
                    "tool_exhaustion",
                    "uncertainty",
                    "safety_concern",
                ],
            },
            "context_summary": {
                "type": "string",
                "maxLength": 1000,
                "description": "Summary of the conversation and what the patient/clinician needs.",
            },
            "idempotency_key": {"type": "string", "format": "uuid"},
        },
        "required": ["patient_id", "escalation_reason", "context_summary", "idempotency_key"],
    },
    side_effect="escalation",
)
def escalate_to_care_manager(...) -> EscalationResult:
    # ...
```

**Why this works.** The four when-to-escalate criteria are explicit, so the agent has a clear signal. The escalation_reason enum supports analytics (which categories drive escalation?). The context_summary gives the care manager what they need to take over. The escalation is the natural termination path when the agent can't proceed safely.

### 9.4 `bash` — the tool Meridian deliberately does NOT have

The team explicitly does not expose a bash or shell tool. The reasoning, recorded in the catalogue's design document:

> A bash tool would be useful for some edge cases (file inspection, log analysis) but its capability scope is unbounded. The model could execute arbitrary commands; mistakes could be catastrophic; security review would block deployment. For each capability that's tempting, we ship a specific bounded tool — `read_log_file(log_name, time_range)`, `fetch_file_metadata(path)` — instead.

The discipline scales: every "could we just give the agent a bash tool?" suggestion is rejected; a specific tool is built instead.

### 9.5 Registry-level policy

Every tool dispatch passes through the registry, which:

- Validates the tenant scope. The care-coordinator at Tenant A cannot fetch_patient from Tenant B.
- Validates the user authorization. A care-coordinator session for a clinician with limited scope cannot escalate to a senior care manager directly.
- Logs the dispatch with full attributes per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md).
- Enforces per-tool rate limits.

### 9.6 The catalogue evolution

In one year of operation, the catalogue evolved:

- Started at 18 tools; trimmed to 12 by deleting overlapping tools (`get_patient` and `lookup_patient` merged into `fetch_patient`).
- Added enum constraints to several arguments after observing the model passing free-text values that didn't match expected categories.
- Rewrote descriptions of three tools after eval surfaced selection accuracy below 90% (model was picking the wrong tool ~10% of the time on ambiguous cases).
- Migrated two tools (`read_log_file`, `fetch_file_metadata`) to a sandboxed MCP server after a near-miss with an in-process read that exposed too much filesystem.

The catalogue is treated as a living artefact, not a one-time setup.

---

## 10. Anti-patterns

### 10.1 "Tool description is engineer-facing"

Description reads "internal patient_v3 wrapper, see also patient_v2 deprecated 2025-Q3." The model doesn't know what to do with this.

**Corrective.** Descriptions are written for the model. State what, when, returns, and what-not-to-use-for.

### 10.2 "Catalogue keeps growing"

The team adds tools whenever a need arises; never deletes. The 47-tool catalogue produces worse model decisions than the 12-tool one.

**Corrective.** Catalogue size discipline. Periodic deletion of unused tools. Cap and enforce.

### 10.3 "Generic bash / any-HTTP tool"

The model has a `bash` or `http_request` tool. The capability scope is unbounded; security review didn't catch it; production incidents follow.

**Corrective.** Specific tools for specific capabilities. No general-purpose execution primitives.

### 10.4 "Side effect without idempotency"

A `send_message` tool sends an SMS. The agent's loop retries on transient failure. The patient gets duplicate texts.

**Corrective.** Idempotency key required. Runtime de-duplicates.

### 10.5 "Error envelope is unstructured"

The tool returns `"Internal error"` on failure. The model can't recover; the repair loop runs to budget exhaustion.

**Corrective.** Structured error envelope per section 5.2 with error_code, message, retryable, suggested_action.

### 10.6 "Argument schema is `options: dict`"

The model passes whatever fields it invents; the tool ignores unknown fields silently; behaviour is silently wrong.

**Corrective.** Specific named arguments with schemas. Strict validation.

### 10.7 "Descriptions don't say what NOT to use the tool for"

Two similar tools have similar descriptions; the model picks at random; selection accuracy is poor.

**Corrective.** Negative space in descriptions. "Do not use for X — use Y instead."

### 10.8 "Tool isn't observed"

Tool calls don't emit spans. When the agent misbehaves, the trace doesn't show what tools were called with what arguments.

**Corrective.** Every tool dispatch emits a span per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md).

---

## 11. Findings (sprint-assignable)

### AGT-TOOL-001 — Severity: Critical
**Finding.** Tool catalogue includes a generic execution primitive (`bash`, `shell`, `python_repl`, or `http_request`).
**Recommendation.** Remove the generic tool; replace with specific bounded tools for each genuine capability need.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-TOOL-002 — Severity: Critical
**Finding.** Side-effecting tools have no idempotency mechanism; retries can double-execute.
**Recommendation.** Idempotency key required per tool per section 6.2; runtime de-duplicates.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-TOOL-003 — Severity: Critical
**Finding.** Agent has a direct-action tool for a high-stakes operation (charge card, delete record, send communication without approval).
**Recommendation.** Propose pattern per section 6.3; agent proposes, human approves, execution is outside the loop.
**Owner.** ai-platform-eng + product, sprint N+1.

### AGT-TOOL-004 — Severity: High
**Finding.** Tool descriptions are engineer-facing; selection accuracy below 85% on eval.
**Recommendation.** Rewrite descriptions per section 3.2 (what / when / returns / what not to use for).
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-TOOL-005 — Severity: High
**Finding.** Argument schemas are loose (free-text where enums or formats apply); model often supplies invalid values.
**Recommendation.** Strict schemas per section 4.3 with enums, formats, constraints.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-TOOL-006 — Severity: High
**Finding.** Error envelopes are unstructured strings; model cannot interpret error and falls into repair loops.
**Recommendation.** Structured error envelope per section 5.2 across all tools.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-TOOL-007 — Severity: High
**Finding.** Tool catalogue > 25 tools; model selection accuracy degraded.
**Recommendation.** Curate per section 7.3; delete overlapping or unused tools; consider splitting into per-agent subsets.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-TOOL-008 — Severity: High
**Finding.** Tool calls are not authorized at the registry; tools enforce tenant scope inconsistently.
**Recommendation.** Authorization at the registry per section 7.1; tools assume authorization is done.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-TOOL-009 — Severity: Medium
**Finding.** Tool registry does not emit per-call spans; trajectory observability is missing tool-call detail.
**Recommendation.** Spans per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md); attributes per section 7.6.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-TOOL-010 — Severity: Medium
**Finding.** New tools are added without eval impact analysis; existing eval regressions are detected late.
**Recommendation.** PR template for new tool requires eval set extension and a re-run of the existing eval.
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### AGT-TOOL-011 — Severity: Medium
**Finding.** Per-agent tool subsets are not implemented; every agent sees the full catalogue.
**Recommendation.** Per-agent subsets per section 7.4; smaller subsets per agent.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-TOOL-012 — Severity: Medium
**Finding.** Tool descriptions lack the "what not to use for" negative space; similar tools are confused.
**Recommendation.** Description audit per section 3.5; add negative space.
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### AGT-TOOL-013 — Severity: Medium
**Finding.** Listy tools have no limit argument or default; large result sets overwhelm context.
**Recommendation.** Default and maximum limit on every list-returning tool per section 4.4.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-TOOL-014 — Severity: Medium
**Finding.** Third-party MCP servers are installed without security review; capability and trust boundaries are unclear.
**Recommendation.** MCP server review process per section 8.5; security review for any installed MCP server.
**Owner.** ai-platform-eng + ai-security, sprint N+4.

### AGT-TOOL-015 — Severity: Low
**Finding.** Tool return payloads include internal IDs and metadata not useful to the model; context is wasted.
**Recommendation.** Return-shape audit per section 5.1; drop fields the model doesn't need.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-TOOL-016 — Severity: Low
**Finding.** Tool description quality is not measured; selection accuracy is observed only when it degrades.
**Recommendation.** Per-tool selection accuracy as an eval metric; report monthly.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-TOOL-017 — Severity: Low
**Finding.** Tool catalogue is not documented for new engineers; tool-design rationale is tribal knowledge.
**Recommendation.** Catalogue design document committed alongside the registry; per-tool rationale in tool metadata.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-TOOL-018 — Severity: Low
**Finding.** Tool schema changes are not coordinated with prompt or eval changes; tool/prompt drift.
**Recommendation.** Tool-schema changes treated as breaking changes with deprecation windows per section 4.5; prompt and eval updated in the same PR.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team designing a tool catalogue from scratch:

- [ ] **Sprint 0 — capability inventory.** What does the agent need to do? List the capabilities (not the tools yet).
- [ ] **Sprint 0 — capability-to-tool mapping.** For each capability, design 1–3 specific tools. Reject generic primitives.
- [ ] **Sprint 1 — naming and descriptions.** Per section 3. Get them reviewed.
- [ ] **Sprint 1 — argument schemas.** Per section 4. Strict, with enums and formats.
- [ ] **Sprint 1 — return shapes.** Per section 5. Consistent envelope.
- [ ] **Sprint 1 — registry.** Build the registry per section 7; tools registered with scopes.
- [ ] **Sprint 2 — idempotency.** Per section 6. Every side-effecting tool has idempotency.
- [ ] **Sprint 2 — propose pattern.** High-stakes side effects go through proposals + human approval.
- [ ] **Sprint 2 — observability.** Spans per tool dispatch.
- [ ] **Sprint 2 — authorization.** Registry-level authorization per [tool-call-authorization.md].
- [ ] **Sprint 3 — eval set.** Per-tool unit tests + tool-with-agent integration eval.
- [ ] **Sprint 3 — selection accuracy.** Measure on the eval set; target > 95% on the catalogue.
- [ ] **Ongoing — curation.** Quarterly catalogue review; delete unused tools; audit descriptions.

For a team with an existing problematic catalogue:

- [ ] **Sprint 0 — audit.** List all tools; for each: how often is it used, what's its selection accuracy, what's its description quality.
- [ ] **Sprint 0 — deletion list.** Tools used < 1% of the time, tools that overlap, tools that shouldn't exist (bash, generic HTTP).
- [ ] **Sprint 1 — deletions.** Remove the deletion list; verify eval doesn't regress.
- [ ] **Sprint 1 — description rewrites.** Rewrite descriptions of low-accuracy tools.
- [ ] **Sprint 2 — schema tightening.** Add enums, formats, constraints to loose schemas.
- [ ] **Sprint 2 — idempotency retrofit.** Add idempotency keys to side-effecting tools that lack them; handle the backward-compat window.
- [ ] **Sprint 3 — error envelope retrofit.** Standardise on the structured envelope.
- [ ] **Sprint 4 — per-agent subsets.** Split the catalogue per agent.

A team that completes this has a tool surface that produces good model decisions at low operational cost. A team that doesn't has an agent whose quality is held back by its tools more than by its model.

---

## 13. References

- [agent-engineering-playbook.md](./agent-engineering-playbook.md) — broader practice; section 4 (tool architecture).
- [agent-loop-design.md](./agent-loop-design.md) — runner that dispatches tool calls.
- [error-and-partial-failure.md](./error-and-partial-failure.md) — model-side handling of tool errors and partial results.
- [memory-engineering.md](./memory-engineering.md) — tools that read/write memory (memory primitives are themselves tools).
- [agent-evals.md](./agent-evals.md) — tool-with-agent eval pattern.
- [agent-observability.md](./agent-observability.md) — trajectory observability that includes tool spans.
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — span shape for tool calls.
- [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — instrumentation for retrieval tools specifically.
- [prompt-engineering/structured-output-engineering.md](../prompt-engineering/structured-output-engineering.md) — schema discipline (similar to argument schemas here).
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/tool-call-authorization.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/tool-call-authorization.md) — the architectural authorization pattern.
- Sibling repo: [ai-architecture-reference-architecture/integration-architecture/tool-call-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/tool-call-architecture.md) — integration patterns for tool-mediated calls (planned doc).
- Sibling repo: [ai-security-reference-architecture/agent-security/](https://github.com/jeremiahredden/ai-security-reference-architecture/tree/main/agent-security) — security depth on tool surface, MCP hardening, prompt injection in tool outputs.
- Model Context Protocol (MCP) — Anthropic specification for inter-process tool servers; reference implementation and ecosystem.
- Anthropic tool use docs, OpenAI function calling docs — provider-specific surface for the tool/function calling capability.
