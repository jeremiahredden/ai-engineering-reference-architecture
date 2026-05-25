# Structured Output Engineering

> **Audience.** Engineers shipping AI features that consume the model's output programmatically (tool calls, JSON parsers, downstream extractors). Tech leads who have seen "the model usually returns valid JSON but sometimes doesn't, and we can't trust the downstream parser." **Scope.** The *engineering* practice for structured output — JSON Schema with tool calls, constrained decoding, output validation with repair loops, failure modes. Pair with [prompts-as-code-discipline.md](./prompts-as-code-discipline.md), [agent-engineering-playbook.md](../agent-engineering/agent-engineering-playbook.md), [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Structured output is the bridge between AI and the rest of the system. The model returns JSON; the downstream code parses; the parsed object drives subsequent logic. Done well, structured output is reliable enough that the consuming code treats it like any other API response. Done poorly, structured output is "usually JSON but sometimes something else" — fragile downstream code; periodic production failures; trust erosion.

The discipline this document codifies: structured output is engineered to be reliable enough that downstream code can depend on it. The pattern combines schema specification, model features (tool calling, constrained decoding), output validation, and bounded repair. Without the pattern, structured output is hope-driven; with it, structured output is contractual.

This document is opinionated about three things:

1. **JSON Schema with tool calling is the production default.** Most modern providers (Anthropic, OpenAI, Google) support tool calling with schema enforcement; the model's output is validated against the schema before return.
2. **Output validation is mandatory.** Even with schema enforcement, validate the output; some failures slip through (escape characters, schema-edge-cases).
3. **Bounded repair loops are the recovery pattern.** When validation fails, re-prompt with the error; bounded retries (2 typical); after exhaustion, fail cleanly.

Structure: (2) the structured output patterns; (3) JSON Schema with tool calling; (4) constrained decoding alternatives; (5) output validation; (6) repair loops; (7) failure modes; (8) integration with the LLM-call wrapper; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The structured output patterns

The patterns for getting reliable structured output from LLMs.

### 2.1 Pattern A: Tool calling with JSON Schema

The model is given a tool definition with a JSON Schema; it returns the tool call with arguments matching the schema. Provider's API validates the schema match.

```json
{
  "tools": [{
    "name": "submit_clinical_recommendation",
    "input_schema": {
      "type": "object",
      "properties": {
        "recommendation": {"type": "string"},
        "confidence": {"type": "string", "enum": ["high", "medium", "low"]},
        "cited_sources": {
          "type": "array",
          "items": {"type": "string"}
        }
      },
      "required": ["recommendation", "confidence", "cited_sources"]
    }
  }]
}
```

The model invokes the tool; the response contains a tool_use block with validated JSON arguments.

**When right.** The default for production structured output in 2026. Most reliable; most provider support.

### 2.2 Pattern B: JSON mode / structured response format

The model is instructed to return JSON; the provider's API enforces JSON-validity (but not specific schema).

```python
response = llm_client.call(
    response_format={"type": "json_object"},
    messages=[...],
)
```

**When right.** When the application needs JSON but the schema is dynamic or hard to express. Simpler than tool calling.

**What it costs.** Doesn't enforce a specific schema; the application validates the schema after parsing.

### 2.3 Pattern C: Constrained decoding

The provider (or a wrapper library) constrains the model's token generation to only produce tokens matching the schema. The model can't produce invalid output.

Examples: Outlines, Instructor (with token-level enforcement), some provider features.

**When right.** When the schema is strict and provider tool-calling doesn't cover the case; or for self-hosted inference.

**What it costs.** May limit the model's flexibility; not all providers support; requires specific library integration.

### 2.4 Pattern D: Free-form with extraction

The model produces free-form output (text or loosely-structured); a separate extraction step parses the output into structured form.

```python
free_form = llm_call(messages)
extracted = extract_with_second_llm(free_form, schema)
```

**When right.** When the model's primary task is free-form (a clinical answer, a written summary) but a structured representation is also needed downstream.

**What it costs.** Two LLM calls; extraction can fail.

### 2.5 The pattern choice

| Workload | Recommended pattern |
|---|---|
| Tool-use agent dispatching actions | A (tool calling with schema) |
| API returning structured data | A or B |
| Self-hosted with strict schema | C (constrained decoding) |
| Free-form answer with structured metadata | D (free-form + extract) |
| Simple structured data with stable schema | A |
| Dynamic / generated schemas | B with post-validation |

Most production workloads use A as the default; B for simpler cases; D for free-form workflows with structured metadata needs.

---

## 3. JSON Schema with tool calling

The detailed pattern.

### 3.1 The schema definition

The JSON Schema defines the expected output structure. Best practices:

- **Required vs optional fields.** Mark required clearly; optional fields are okay.
- **Enums for categorical fields.** `"enum": ["high", "medium", "low"]` is more reliable than free-form strings.
- **Type constraints.** Strings, numbers, booleans, arrays, objects with their own schemas.
- **Descriptions.** Each field has a description; the model uses the descriptions to understand what to produce.
- **Examples (optional).** Example values can be included; supports the model's understanding.

### 3.2 The schema as code

The schema lives in the team's repository alongside the prompt:

```python
class ClinicalRecommendationSchema(BaseModel):
    recommendation: str = Field(description="The clinical recommendation, concise")
    confidence: Literal["high", "medium", "low"] = Field(description="Model's confidence")
    cited_sources: list[str] = Field(description="Source IDs cited in the recommendation")
    requires_escalation: bool = Field(description="Whether the case requires human escalation")
```

Pydantic, JSON Schema, or other representations work; the key is that the schema is version-controlled.

### 3.3 The schema versioning

Per [prompts-as-code-discipline.md](./prompts-as-code-discipline.md), schemas are versioned. Schema changes:

- **Additive (new optional field).** Minor version bump; backward-compatible.
- **Breaking (renamed field, type change).** Major version bump; coordinated migration.

Downstream consumers depend on the schema; changes go through review.

### 3.4 The tool naming for non-action use

When using tool calling for structured output (not actual tool actions), name the "tool" semantically:

- `submit_clinical_recommendation` — the model is "submitting" its recommendation.
- `extract_patient_summary` — the model is "extracting" structured data.
- `classify_query` — the model is "classifying."

The name conveys intent; the model produces appropriate output.

### 3.5 The model's reliability with schema

In 2026, frontier models (Claude Opus, GPT-5, Gemini Pro) have strong schema-adherence. Reliability ~98-99% for well-defined schemas. The 1-2% failures need handling per repair loops (section 6).

Mid-tier models (Sonnet, GPT-4o-mini) are somewhat less reliable; ~95%. Cheap-tier models can be ~90%. Reliability varies by schema complexity.

### 3.6 The provider differences

- **Anthropic.** Tool calling with `input_schema` (JSON Schema); strong schema adherence.
- **OpenAI.** Tool calling with `parameters` (JSON Schema); also supports `response_format: json_object` and `json_schema`.
- **Google.** Function calling with schemas.

The LLM-call wrapper (per [llm-call-instrumentation.md](./llm-call-instrumentation.md)) abstracts provider differences; consumers see a uniform structured-output API.

---

## 4. Constrained decoding alternatives

When tool calling isn't enough.

### 4.1 The constrained decoding pattern

Libraries that constrain the model's token-level generation:

- **Outlines.** Open-source; constrains generation to match a regex or JSON Schema; works with multiple model backends.
- **Instructor.** Python library for structured output with validation; works on top of provider APIs.
- **LMQL.** Query language for constrained LLM generation.

**When right.**
- Self-hosted inference (the team controls token generation).
- Schemas the provider's tool calling doesn't support well.
- Deterministic constraints (the output cannot violate the schema, by construction).

**Trade-off.** Library complexity; potentially restrictive on the model's flexibility.

### 4.2 The instructor pattern

Per the Instructor library:

```python
import instructor
from pydantic import BaseModel

class Summary(BaseModel):
    title: str
    bullets: list[str]
    confidence: float

client = instructor.patch(llm_client)
summary = client.chat.completions.create(
    model="claude-opus-4-7",
    messages=[...],
    response_model=Summary,
)
```

The library handles retries and validation transparently.

### 4.3 The selection between Pattern A and Pattern C

| Workload | Pattern A (tool calling) | Pattern C (constrained decoding) |
|---|---|---|
| Hosted provider | ✓ | Less common |
| Self-hosted | Possible | ✓ |
| Strong provider schema support | ✓ | Less needed |
| Complex schemas | ✓ | ✓ |
| Latency-critical | Slight overhead | Token-level overhead |

Most teams use A. C is a specialized choice.

---

## 5. Output validation

Even with schema enforcement, validate the output.

### 5.1 Why validate

Provider schema enforcement isn't perfect:

- Escape characters in strings can produce subtle invalid JSON.
- Edge-case types (very long strings, deeply-nested objects) sometimes fail.
- Provider bugs occasionally produce schema violations.

The 1-2% failure rate (per section 3.5) needs handling.

### 5.2 The validation step

```python
def call_with_structured_output(messages, schema):
    response = llm_client.call(messages, tools=[schema_to_tool(schema)])
    
    if not response.has_tool_use:
        raise StructuredOutputError("No tool use returned")
    
    raw_output = response.tool_use.input
    
    try:
        validated = schema.parse_obj(raw_output)  # Pydantic example
    except ValidationError as e:
        raise StructuredOutputValidationError(raw_output, e)
    
    return validated
```

Validation is deterministic; the consumer either gets a validated object or a clearly-classified error.

### 5.3 The validation scope

What to validate:

- **Schema compliance.** Required fields present; types correct; enums valid.
- **Domain constraints.** Beyond schema — business rules (e.g., "patient_id must exist", "score must be 0-1").
- **Cross-field consistency.** Multi-field constraints (e.g., "if escalation_required, escalation_reason must be present").

Pure schema validation catches the basic class; domain validation catches the rest.

### 5.4 The validation layer

Validation can be:

- **Pydantic models.** Common Python pattern; declarative; supports custom validators.
- **JSON Schema validators.** Schema-driven; supports any JSON Schema feature.
- **Custom validators.** For domain-specific logic.

Most teams use Pydantic for the application; JSON Schema for the API surface.

### 5.5 The validation observability

Validation failures are tracked:

- Per-feature failure rate.
- Per-schema failure rate.
- Failure-class distribution (schema vs domain vs consistency).

Spikes in failure rate are investigated; may indicate model regression or schema mismatch.

---

## 6. Repair loops

When validation fails, the bounded retry pattern.

### 6.1 The repair pattern

```python
def call_with_repair(messages, schema, max_attempts=3):
    for attempt in range(max_attempts):
        try:
            response = llm_client.call(messages, tools=[schema_to_tool(schema)])
            return validate(response.tool_use.input, schema)
        except StructuredOutputValidationError as e:
            if attempt == max_attempts - 1:
                raise
            messages = append_repair_message(messages, e)
    
def append_repair_message(messages, error):
    return messages + [
        {"role": "user", "content": f"Your previous response had a validation error: {error}. Please re-submit with the correct schema."}
    ]
```

The model gets the error and another chance. Bounded retries prevent unbounded cost / latency.

### 6.2 The bounded retry count

Typical: 2 retries (3 total attempts).

- After 3 attempts: the failure is structural; further retries are unlikely to succeed.
- The call returns a clean error; downstream code handles the failure.

### 6.3 The cost of repair

Each repair attempt is a new LLM call. For a 3-attempt-limit:

- Best case: 1 call (typical).
- Worst case: 3 calls (rare).
- Average: ~1.02-1.05 calls (very low repair rate in production).

Cost is bounded; the repair pattern is affordable.

### 6.4 The repair telemetry

Per attempt:

- `ai.llm.repair_attempt: 0 | 1 | 2`
- `ai.llm.repair_reason: "missing_field" | "type_mismatch" | etc.`

Aggregate:

- Per-feature repair rate (target < 1%).
- Per-schema repair rate.

High repair rates indicate the schema or prompt needs revision.

### 6.5 The repair fallback

After repair exhaustion, the consumer needs a path:

- For non-critical features: return a degraded response ("I couldn't produce a structured answer; here's free-form").
- For critical features: escalate to human; refuse cleanly.

The fallback is per-feature; documented per [fallback-patterns.md](../reliability-engineering/fallback-patterns.md).

---

## 7. Failure modes

The specific failure modes for structured output.

### 7.1 The schema violation

Model returns output that doesn't match the schema.

Examples:
- Missing required field.
- Wrong type for a field.
- Enum value not in the allowed set.

Handling: validation catches; repair loop attempts to fix.

### 7.2 The truncation

Model's output exceeds max_tokens; output is truncated mid-JSON; parsing fails.

Examples:
- Long array of items; truncation cuts mid-item.
- Long string field; truncation cuts mid-string.

Handling: validation catches; repair with higher max_tokens, or reduce expected output size.

### 7.3 The escape character issue

Strings with special characters (newlines, quotes, backslashes) sometimes produce subtle invalid JSON.

Examples:
- String contains unescaped quote.
- String contains literal newline.

Handling: provider often handles this correctly, but edge cases exist. Robust JSON parsing (with tolerance for some encoding variations) helps.

### 7.4 The hallucinated field

Model adds a field not in the schema.

Examples:
- Extra "notes" field the schema didn't declare.

Handling: strict validation rejects; lenient validation accepts and discards. The choice is workload-specific.

### 7.5 The semantically-wrong field

Output matches the schema but the values are semantically wrong.

Examples:
- `confidence: "high"` when the answer is actually low-confidence.
- `cited_sources: ["fake_source_id"]` when the source doesn't exist.

Handling: domain validation per section 5.3. Citation validation per [eval-of-rag.md](../eval-engineering/eval-of-rag.md) section 5.

### 7.6 The format-instability

Model's structured output is inconsistent across calls (slight format variations).

Examples:
- Sometimes returns numeric IDs; sometimes string IDs.
- Sometimes arrays of objects; sometimes arrays of strings.

Handling: tighter schema specification; prompt clarification; model-tier change.

### 7.7 The provider-side bug

Provider's schema enforcement has a bug; output doesn't match schema but provider doesn't detect.

Handling: client-side validation catches; report the bug to the provider.

---

## 8. Integration with the LLM-call wrapper

Structured output integrates with the wrapper.

### 8.1 The wrapper's structured-output API

The wrapper exposes a structured-output method:

```python
def call_structured(
    self,
    provider: str,
    model: str,
    messages: list,
    schema: Type[BaseModel],
    *,
    prompt_version: str,
    context: CallContext,
    max_repair_attempts: int = 2,
) -> BaseModel:
    """Call the LLM and return a validated structured response."""
```

The wrapper handles:
- Tool definition from the schema.
- Provider-specific API translation.
- Validation.
- Repair loop.
- Trace emission.
- Cost tracking.

### 8.2 The trace span

Each structured-output call's trace includes:

- The underlying LLM call(s) (one per repair attempt).
- Validation attribute (`ai.structured.validation: passed | failed`).
- Repair attribute (`ai.structured.repair_attempts: N`).
- Schema version (`ai.structured.schema_version: ...`).

The trace supports diagnosis.

### 8.3 The cost attribution

Repair attempts are separately attributed:

- `ai.cost.usd.attempt_0: <cost>`
- `ai.cost.usd.attempt_1: <cost>` (if repair fired)
- `ai.cost.usd.total: <sum>`

Visibility into repair cost informs whether the cost is acceptable.

### 8.4 The failure handling

If the call exhausts repair attempts:

- Wrapper raises `StructuredOutputExhausted`.
- The trace records the failure.
- The consumer handles per the fallback pattern.

### 8.5 The integration with the agent loop

Per [agent-engineering-playbook.md](../agent-engineering/agent-engineering-playbook.md), the agent loop calls tools (each with a schema). The agent's tool-call mechanism uses structured output:

- The agent decides to call a tool.
- The tool's input schema is the structured-output schema.
- The wrapper invokes the structured-output flow.
- The tool receives validated input.

The agent and structured-output are aligned; they're the same mechanism.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The structured-output use cases

In the Care Coordinator:

- **Classifier worker.** Returns a structured classification (case_class, confidence, dispatch_target).
- **Clinical-knowledge worker.** Returns recommendations with cited sources.
- **Drafting worker.** Returns drafts with structured metadata (recipient, template, content).
- **Side-effect tools.** Input schemas (draft_patient_message, propose_specialist_referral) per [tool-call-authorization.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/tool-call-authorization.md).

Each uses Pattern A (tool calling with JSON Schema).

### 9.2 The classifier schema

```python
class ClassificationOutput(BaseModel):
    case_class: Literal["clinical_protocol", "drug_interaction", "general_inquiry", "coordination_task", "out_of_scope"]
    confidence: float = Field(ge=0.0, le=1.0)
    dispatch_target: Optional[str] = None
    reasoning: str = Field(max_length=200)
```

The schema is enforced via Anthropic's tool calling. The classifier prompt instructs the model to invoke `submit_classification` with the schema.

### 9.3 The validation in practice

Production validation results:

- Schema compliance: 99.4% pass rate.
- Domain validation (confidence in 0-1, valid case_class): 99.6% pass rate (some additional catches beyond schema).
- Repair attempts triggered: 0.8% of calls.
- Repair success rate: 92% (most repairs succeed on attempt 2).
- Exhausted repairs: 0.06% of calls.

The structured-output reliability is high; downstream consumers can depend on it.

### 9.4 The 2026-Q1 schema-failure spike

In 2026-Q1, validation failure rate spiked from 0.6% to 4.2%.

Investigation:
- Per-schema breakdown: drafting worker schema was the cause.
- Drafting schema had been updated to add a new required field; the prompt wasn't updated to mention the new field.
- Fix: prompt updated; eval-validated.
- Failure rate returned to baseline.

The spike-investigation followed the trace-debugging workflow per [debugging-from-traces.md](./debugging-from-traces.md).

### 9.5 The repair loop in production

Repair attempts are tracked:

- Average attempts per call: 1.02.
- 99th percentile attempts: 2.
- Maximum attempts seen: 3 (a small number of cases that exhaust).

The bounded repair is operationally efficient.

### 9.6 The cost impact

Structured-output overhead:

- Per-call cost: ~$0.0003 average (one tool-call response).
- Repair overhead: < 1% of total cost (rare repairs).

Cost is small; the engineering ROI is high (reliable downstream consumption).

### 9.7 The platform discipline

- All structured-output via tool calling.
- Schemas in version control per [prompts-as-code-discipline.md](./prompts-as-code-discipline.md).
- Validation mandatory.
- Repair loop bounded (max 2 retries).
- Per-feature failure rate tracked.

---

## 10. Anti-patterns

### 10.1 "JSON in the prompt and hope"

Prompt says "return JSON"; no schema enforcement; no validation. Output is often valid but sometimes not; downstream code fails periodically.

**Corrective.** Tool calling with schema per section 3; validation per section 5.

### 10.2 "Validation absent"

Model returns; output is used directly; no validation; consumers trust the output. Edge-case failures cascade.

**Corrective.** Mandatory validation per section 5.

### 10.3 "Unbounded repair"

Repair loop retries until success; cost and latency unbounded.

**Corrective.** Bounded repair (2-3 attempts) per section 6.2.

### 10.4 "Schema in prompt only"

The schema is described in the prompt as text; not enforced; the model produces approximately-conforming output.

**Corrective.** Tool calling with formal schema per section 3.

### 10.5 "No domain validation"

Schema validation passes; the values are semantically wrong; downstream consumes bad data.

**Corrective.** Domain validation per section 5.3.

### 10.6 "Schema not versioned"

Schema changes ship without version coordination; downstream parsers break.

**Corrective.** Schema as versioned artifact per section 3.3.

### 10.7 "Failure rate not tracked"

Validation failures occur; no monitoring; team learns about issues from user reports.

**Corrective.** Per-feature failure rate per section 5.5.

### 10.8 "Repair exhaustion silent"

Repair exhausts; the wrapper raises; consumer catches and continues with empty data; the failure is invisible.

**Corrective.** Repair exhaustion handled deliberately per section 6.5.

---

## 11. Findings (sprint-assignable)

### STRUCT-001 — Severity: Critical
**Finding.** Structured output via "JSON in prompt"; no schema enforcement.
**Recommendation.** Tool calling with schema per section 3.
**Owner.** ai-platform-eng, sprint N+1.

### STRUCT-002 — Severity: Critical
**Finding.** No output validation; consumers trust model output directly.
**Recommendation.** Mandatory validation per section 5.
**Owner.** ai-platform-eng, sprint N+1.

### STRUCT-003 — Severity: High
**Finding.** Repair loop unbounded; cost and latency runaway.
**Recommendation.** Bounded repair per section 6.2.
**Owner.** ai-platform-eng, sprint N+2.

### STRUCT-004 — Severity: High
**Finding.** Schemas not versioned; downstream breakage on schema changes.
**Recommendation.** Schema as versioned artifact per section 3.3.
**Owner.** ai-platform-eng, sprint N+2.

### STRUCT-005 — Severity: High
**Finding.** Failure rate not tracked; issues discovered by user reports.
**Recommendation.** Per-feature failure rate per section 5.5.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### STRUCT-006 — Severity: High
**Finding.** Domain validation absent; schema-valid but semantically-wrong output flows downstream.
**Recommendation.** Domain validation per section 5.3.
**Owner.** ai-platform-eng, sprint N+2.

### STRUCT-007 — Severity: High
**Finding.** Repair exhaustion handled inconsistently; some consumers silently use empty data.
**Recommendation.** Deliberate handling per section 6.5.
**Owner.** ai-platform-eng, sprint N+2.

### STRUCT-008 — Severity: Medium
**Finding.** Trace observability for structured output incomplete.
**Recommendation.** Trace attributes per section 8.2.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### STRUCT-009 — Severity: Medium
**Finding.** Wrapper doesn't expose a structured-output API; consumers re-implement.
**Recommendation.** Structured-output API per section 8.1.
**Owner.** ai-platform-eng, sprint N+3.

### STRUCT-010 — Severity: Medium
**Finding.** Per-schema failure rate not tracked; per-schema issues hide in aggregate.
**Recommendation.** Per-schema metric per section 5.5.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### STRUCT-011 — Severity: Medium
**Finding.** Schema descriptions thin; model produces less-reliable output.
**Recommendation.** Rich descriptions per section 3.1.
**Owner.** ai-platform-eng, sprint N+3.

### STRUCT-012 — Severity: Medium
**Finding.** Constrained-decoding option not evaluated for self-hosted workloads.
**Recommendation.** Evaluate per section 4.
**Owner.** ai-platform-eng, sprint N+4.

### STRUCT-013 — Severity: Medium
**Finding.** Truncation failure mode not handled; mid-JSON cuts produce parse errors.
**Recommendation.** Max-tokens calibration; repair on truncation per section 7.2.
**Owner.** ai-platform-eng, sprint N+4.

### STRUCT-014 — Severity: Medium
**Finding.** Hallucinated-field handling inconsistent; some validators strict, some lenient.
**Recommendation.** Per-schema strictness decision documented.
**Owner.** ai-platform-eng, sprint N+4.

### STRUCT-015 — Severity: Medium
**Finding.** Cost attribution for repair attempts absent; high-repair cases invisible in cost dashboards.
**Recommendation.** Per-attempt cost per section 8.3.
**Owner.** ai-platform-eng + finops, sprint N+4.

### STRUCT-016 — Severity: Low
**Finding.** Schema-as-prompt-only fallback for older provider APIs.
**Recommendation.** Migrate to tool calling per section 3 as providers update.
**Owner.** ai-platform-eng, sprint N+5.

### STRUCT-017 — Severity: Low
**Finding.** Cross-call schema changes can break agent loops that depend on prior outputs.
**Recommendation.** Schema versioning per section 3.3 with consumer notification.
**Owner.** ai-platform-eng, sprint N+5.

### STRUCT-018 — Severity: Low
**Finding.** Documentation thin; new engineers don't understand structured-output patterns.
**Recommendation.** Documentation alongside the wrapper.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team adding structured output discipline:

- [ ] **Sprint 0 — design.** Identify structured-output use cases; choose patterns.
- [ ] **Sprint 1 — schema definitions.** Schemas in version control; Pydantic or equivalent.
- [ ] **Sprint 1 — tool calling integration.** Wrapper supports structured output via tool calling.
- [ ] **Sprint 2 — validation.** Mandatory schema + domain validation.
- [ ] **Sprint 2 — repair loop.** Bounded repair (2 attempts default).
- [ ] **Sprint 3 — observability.** Per-feature, per-schema failure rate; trace attributes.
- [ ] **Sprint 3 — repair exhaustion handling.** Per-feature fallback decisions documented.
- [ ] **Sprint 4 — cost attribution.** Per-attempt cost; aggregate per-feature.
- [ ] **Sprint 4 — schema versioning discipline.** Schema changes through PR review; downstream notification.
- [ ] **Sprint 5 — advanced patterns.** Constrained decoding for specific use cases; free-form + extraction patterns.
- [ ] **Ongoing — discipline.** Schemas as code; validation mandatory; repair bounded; failure rate monitored.

A team that completes this sequence has structured output reliable enough to depend on. A team that ships JSON-in-prompt patterns pays in periodic production failures.

---

## 13. References

- This repo: [prompt-engineering/prompts-as-code-discipline.md](./prompts-as-code-discipline.md) — schema as code.
- This repo: [prompt-engineering/prompt-versioning.md](./prompt-versioning.md) — schema versioning.
- This repo: [agent-engineering/agent-engineering-playbook.md](../agent-engineering/agent-engineering-playbook.md) — agent loop's tool-call mechanism.
- This repo: [agent-engineering/agent-loop-design.md](../agent-engineering/agent-loop-design.md) — model-response parsing layer.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — wrapper integration.
- This repo: [reliability-engineering/fallback-patterns.md](../reliability-engineering/fallback-patterns.md) — repair-exhaustion fallback.
- This repo: [eval-engineering/eval-of-rag.md](../eval-engineering/eval-of-rag.md) — citation validation patterns.
- Anthropic, OpenAI, Google function-calling / tool-calling documentation.
- Outlines, Instructor, LMQL libraries.
- JSON Schema specification; Pydantic documentation.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/tool-call-authorization.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/tool-call-authorization.md) — tool registry where structured-output schemas live.
