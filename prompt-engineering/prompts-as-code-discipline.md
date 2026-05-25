# Prompts-as-Code Discipline

> **Audience.** Engineers and tech leads whose team's prompts live as inline strings in code or as edited cells in someone's notebook. Anyone who has been asked "which version of the prompt was running yesterday" and could not answer. **Scope.** The *engineering* practice of pulling prompts out of inline strings into a versioned store with ownership, change control, and CI integration. Pair with [prompt-versioning.md](./prompt-versioning.md) for the version-lifecycle pattern. Not the prompt-writing-technique (vendor docs cover that); not the architectural choice of prompt assembly patterns (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `context-and-prompt-architecture/`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The prompt is software. It is the largest single behavioral lever a team has over an AI system. It fails in production. It has regressions. It has backwards-compatibility concerns with downstream parsers, evals, and chained calls. It needs ownership, review, deployment, and rollback.

Most production teams in 2026 still treat prompts the way teams in 2010 treated SQL queries — embedded as inline strings in application code, edited by whoever happens to be working on the file, with no versioning, no review, no test suite, no rollback path. The team eventually arrives at the moment they cannot tell which version of the prompt was in production when a quality regression happened. By then, the cost of refactoring to a versioned prompt store is much higher than it would have been on day one.

This document is the on-ramp for that refactor. It is not opinionated about which prompt store technology to use — Git is fine, a database table is fine, a vendor prompt-registry (LangSmith Prompts, Braintrust Prompts, Helicone Prompts) is fine. It is opinionated about the *discipline*: prompts live as named artifacts, versioned, owned, deployed, reviewable.

This document is opinionated about three things:

1. **Prompts are pulled out of inline strings.** This is the load-bearing refactor. Application code references prompts by ID; the prompt content lives in the store. Inline strings are lint-rejected for production prompts.
2. **Each prompt has an owner.** A person or team accountable for the prompt's quality, evolution, and deprecation. Without an owner, the prompt degrades.
3. **Prompt changes go through the same CI as code changes.** Eval-gated per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md). Reviewer approval required. Deployable as a release artifact. Rollbackable independently.

Structure: (2) the four maturity stages most teams move through; (3) the prompt-store options; (4) the prompt artifact structure; (5) ownership; (6) the change-control workflow; (7) the migration path from inline strings; (8) integration with the LLM-call wrapper; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The four maturity stages

Most teams move through four stages, often without realizing they are in one. Naming the stage clarifies the next move.

### 2.1 Stage 0: Notebook strings

Prompts live as strings in Jupyter notebooks or in someone's local Python script. Production deployment is "copy the string into the application's code." Nobody can confidently say what prompt is in production right now.

**Signs you are here.** No one can answer "which prompt is running" without reading source code. Prompts have no version. Two engineers ship two different versions of "the same" prompt unknowingly.

**Move to stage 1.** Pull prompts out of notebooks into named artifacts inside the application's source tree.

### 2.2 Stage 1: Source-tree artifacts

Prompts live as files (e.g., `prompts/supervisor.txt` or `prompts/clinical_knowledge.py` returning a string). They are in version control. Application code imports them by name.

**Signs you are here.** Prompts have file names and can be searched. They are committed alongside code changes that use them. There is no separate versioning beyond the git history.

**Move to stage 2.** Add explicit versioning so a release can pin a specific prompt version independently of the application's git SHA.

### 2.3 Stage 2: Versioned artifacts in source

Prompts have explicit versions (e.g., `prompts/supervisor/v2.4.1.md` plus a manifest). Application code references the prompt by name + version. Multiple versions of the same prompt can coexist; the application code points at one.

**Signs you are here.** Prompts have semver-style versions. Rollback is possible without a full code rollback. Prompts have basic ownership (the file's git blame is the de facto owner).

**Move to stage 3.** Move prompts to a runtime-updateable store so prompt changes do not require a code deploy.

### 2.4 Stage 3: Versioned artifacts in runtime store

Prompts live in a runtime store (database, vendor prompt registry) that the application reads at runtime (or near-runtime via cache). Prompt changes deploy independently of code. Multiple versions are available at runtime; routing logic picks the version.

**Signs you are here.** Prompt changes do not require a code deploy. The prompt store has its own access controls and audit. CI / eval pipelines run on prompt changes the same way they run on code changes.

**There is no stage 4.** Stage 3 is the destination. Stages beyond this (vendor-specific prompt management features) are tooling choices that do not change the fundamental shape.

### 2.5 What stage are you at?

Honest assessment usually places teams one stage behind where they think they are. Notebook strings persist longer than teams realize; source-tree artifacts often lack the explicit versioning that makes them stage 2 rather than stage 1.

The discipline of this document is appropriate at any stage above 0. Stage 1 is the minimum viable; stage 3 is the destination for production AI systems with meaningful prompt evolution.

---

## 3. Prompt-store options

The store is where prompts live. Choose deliberately; the choice influences (but does not determine) the discipline.

### 3.1 Git-as-store

Prompts live in a Git repository as files. The repository is the source of truth. The application reads prompts at deploy time (build-time bundling) or at runtime (reading from a sidecar that syncs from Git).

**Pros.** Free. Familiar. Every engineer already knows Git workflow. PR review, history, blame, branches — all already in place.

**Cons.** Prompt deploys are coupled to git deploys (or require build-time bundling that adds a step). Multi-language teams have to read the same Git repository from different languages.

**When to use.** Small-to-medium prompt sets. Single-language platforms. Teams that already have strong Git discipline.

### 3.2 Database-as-store

Prompts live in a relational or document database (Postgres, DynamoDB, Mongo). A simple schema: prompt-name, version, content, metadata, created-at. The application reads at runtime (often through a cache).

**Pros.** Runtime-updateable. Multi-language out of the box. Easy to add metadata (owner, tags, deprecation status). Easy to query (which prompts use which model).

**Cons.** Requires building the change-control workflow (the database does not naturally support PR review). Versioning is a schema decision; if not designed for it, retrofits are messy.

**When to use.** Multi-language platforms. Teams comfortable building the change-control workflow around the database. Workloads where prompt deploys must be decoupled from code deploys.

### 3.3 Vendor prompt registry

LangSmith Prompts, Braintrust Prompts, Helicone Prompts. The vendor provides a prompt store with versioning, editing UI, integration with their broader eval / observability tooling.

**Pros.** Out-of-the-box editing UI for non-engineers. Integrated with the vendor's other AI tooling. Versioning built-in.

**Cons.** Vendor lock-in (prompts in the vendor's format / API). Coupling between prompt store and the vendor's other tooling. Per-prompt or per-API-call pricing.

**When to use.** Teams already deeply invested in the vendor's ecosystem. Teams where non-engineer prompt authoring (product managers, domain experts) is a real workflow.

### 3.4 Hybrid (Git + database)

Prompts authored in Git (source of truth), synced to a database for runtime serving. Common for teams that want Git's review workflow plus runtime decoupling.

**Pros.** Git's discipline + runtime-decoupled serving. Cache invalidation via the sync pipeline.

**Cons.** Sync pipeline is its own thing to operate. Two sources of truth if the sync ever drifts.

**When to use.** Teams with strong Git discipline that want runtime decoupling. The Meridian Care Coordinator pattern.

### 3.5 The decision matters less than the discipline

All four options support the discipline. The discipline (named artifacts, versioning, ownership, CI integration) is what makes the team's prompts engineerable. The store is implementation.

Most teams choose Git-as-store for the first 6–12 months and graduate to hybrid or database when prompt-deploy frequency outpaces code-deploy frequency.

---

## 4. The prompt artifact structure

Each prompt is a structured artifact, not just a string. The structure makes the prompt reviewable, deployable, and queryable.

### 4.1 The minimal artifact

```yaml
name: care_coordinator_supervisor
version: 2.4.1
owner: ai-platform-eng
deprecation: null
status: active

description: |
  System prompt for the Care Coordinator supervisor agent. Handles
  user-facing dispatch and consolidation; not for direct clinical
  reasoning (that uses care_coordinator_clinical_knowledge).

intended_model: claude-opus-4-7
context_window_target: 8000   # tokens
expected_output_class: tool_call_or_final_answer

content: |
  You are the supervisor for the Meridian Care Coordinator. You receive
  questions from clinical staff and decide how to route them...

  (the full prompt text)

variables:
  - name: tenant_id
    description: The hospital tenant identifier
    required: true
  - name: user_role
    description: The clinical role of the user (rn, ma, physician, etc.)
    required: true
  - name: recent_history
    description: The last N turns of conversation, summarized
    required: false

depends_on:
  retrievers: [clinical-guidelines, tenant-protocols]
  tools: [dispatch_classifier, dispatch_clinical_knowledge,
          dispatch_drafting, dispatch_query_rewriter,
          consult_drug_interaction_graph, finalize_answer]
  prompts: []   # other prompts this one references or invokes

eval_suite_refs:
  - clinical_golden_set
  - conversational_subset
```

The fields:

- **name, version.** The identity. Together they form the unique reference.
- **owner.** Accountable team or person. Not optional.
- **deprecation, status.** The lifecycle state. See [prompt-versioning.md](./prompt-versioning.md).
- **description.** What the prompt is for, what it is not for.
- **intended_model, context_window_target.** The model and context size the prompt was designed against. Used as guardrails — invoking the prompt with a different model produces a warning.
- **expected_output_class.** What the prompt is supposed to produce. Used by output validation.
- **content.** The actual prompt text.
- **variables.** The substitution variables the prompt expects. Schema-validated at invocation time.
- **depends_on.** What downstream components the prompt assumes (retrievers, tools, other prompts). When a referenced component changes, the prompt may need a corresponding update.
- **eval_suite_refs.** Which eval suites cover this prompt. Used to determine which evals to run on changes.

### 4.2 What goes in the content

The content field holds the prompt text with placeholder markers for variable substitution:

```
You are the supervisor for the Meridian Care Coordinator at hospital {{tenant_id}}.
You are speaking with a {{user_role}}.

Recent conversation summary:
{{recent_history}}

Your job is...
```

The variable substitution is done by the prompt-assembly layer (not by the store). The store holds the template; the assembler renders it.

The discipline: variables are explicitly declared in the artifact's `variables` section. Templates that reference undeclared variables fail at validation time, not at runtime.

### 4.3 What does NOT go in the artifact

- **Model parameters (temperature, max_tokens).** Those are call-site decisions, not prompt artifacts.
- **The assembled messages list.** The artifact holds the system prompt (and optionally few-shot examples); the user-message and assistant-message turn structure is assembled at call time.
- **Tool definitions.** Tools come from the tool registry per [tool-call-authorization.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/tool-call-authorization.md). The prompt references which tools it expects to be available; it does not embed tool schemas.
- **Retrieved content.** Retrieval results are injected at call time; the prompt has a placeholder.

The discipline: the prompt is the *durable* part; everything that varies per-call is parameterized.

---

## 5. Ownership

The single biggest reason prompts decay over time is unclear ownership. The discipline:

### 5.1 Every prompt has a named owner

Owner is a team (preferred) or an individual. The owner is accountable for:
- Reviewing changes.
- Maintaining the eval coverage.
- Responding to quality regressions.
- Deciding when to deprecate the prompt.

The owner is recorded in the artifact. CI checks the owner against an allowlist; orphan prompts (owner removed without re-assignment) are flagged.

### 5.2 Ownership boundaries

For the Care Coordinator: ai-platform-eng owns the supervisor and classifier prompts; clinical-knowledge-engineering owns the clinical-knowledge-worker prompt; drafting-team owns the drafting-worker prompt. Specialization matches the prompt's domain.

The owning team has commit authority on their prompts (with code review still required). PR review by the owning team is the gate.

### 5.3 Cross-team prompts

Some prompts cross team boundaries (a shared base prompt that multiple workers extend). The pattern:
- The base prompt has one owner (often platform).
- The extending prompts each have their own owner.
- Changes to the base prompt require notice to the extending-prompt owners.

This is the same dependency-management discipline as any shared library has. Treat prompts as code, including for the dependencies between them.

### 5.4 Orphan-prompt detection

Quarterly: scan the prompt store for prompts whose owner is no longer on the team, whose eval suite has not been updated, whose content has not been touched in > 1 year. These are candidates for retirement or for re-assignment.

The detection is part of the platform discipline. Without it, prompts accumulate without ongoing accountability.

---

## 6. The change-control workflow

Prompt changes are software changes. They go through CI, review, and deployment with the same discipline as code.

### 6.1 The PR workflow

A change to a prompt is a PR:

1. **Author opens PR.** PR includes the new prompt version (or revision of an existing version).
2. **CI runs.**
   - Lint: artifact schema validation; variable references match declarations; content is non-empty; depends_on references resolve.
   - Fast eval: the prompt's eval suite (subset for per-PR speed) runs against the new prompt; pass-rate gate per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) section 5.
   - Cost regression check: estimated per-call cost for the new prompt is compared against the baseline; >20% increase requires sign-off.
3. **Reviewer approves.** Owner-team reviewer is required. For high-stakes prompts (clinical, financial), additional security-eng or domain-expert review is required.
4. **Merge.** The new prompt version is staged for deploy.
5. **Deploy.** Per the prompt-store deployment pattern (build-time bundling, sync pipeline, runtime store update).
6. **Canary.** If the prompt change is a version upgrade (vs a hotfix), traffic is canaried per [cicd-and-eval-gates/canary-rollouts.md](../cicd-and-eval-gates/) (coming).
7. **Full deploy or rollback.** Based on canary signal.

### 6.2 Hotfixes

A prompt-only hotfix (no code change required) deploys faster than a full release. The pattern:
- The hotfix is still a PR with the same gates (eval, review).
- The deploy is direct to production (no canary) when the hotfix is urgent (e.g., production quality regression).
- The hotfix is followed by a normal release that adopts the hotfix into the regular versioning.

Hotfixes are the exception, not the rule. Frequent hotfixes are a signal of insufficient pre-deploy testing.

### 6.3 The eval-gate discipline

A prompt change without an eval pass is rejected. This is the load-bearing CI control. The eval suite is the contract; the prompt change is the proposed implementation; the eval gate is the test.

Some prompt changes intentionally degrade some cases (a tightening of refusal behavior may drop pass-rate on cases that were previously too lenient). For these, the eval-gate override pattern per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) section 5.4 applies — explicit override with reviewer attestation.

### 6.4 Reviewer training

Reviewers of prompt PRs need different skills than reviewers of code PRs. The discipline:
- Reviewers understand the prompt's intent (the description field).
- Reviewers can read the diff and reason about likely behavior changes.
- For high-stakes prompts, reviewers consult the eval results before approving.
- Domain experts (clinicians for clinical prompts, compliance experts for compliance prompts) are reviewers for their domain.

Some teams build a "prompt reviewer" track distinct from the code reviewer track. This is good when the prompt review skill is genuinely different — clinical prompt review benefits from clinical expertise that not every code reviewer has.

---

## 7. The migration path from inline strings

Most teams start at stage 0 or stage 1 and need to migrate. The migration is a structured project, not a sweeping refactor.

### 7.1 Sprint 1: Inventory and pick the first prompt

Inventory every production prompt. Catalog by location, owner (de facto), usage frequency, recency of change. Pick the *most-changed* prompt as the first to migrate — the value of the discipline is highest where change is highest.

### 7.2 Sprint 1: Migrate the first prompt

- Pull the inline string out into an artifact file.
- Add the artifact metadata (owner, description, variables, depends_on).
- Replace the inline string in code with a reference to the artifact.
- Verify behavior unchanged (existing eval suite passes).
- Land the change.

### 7.3 Sprint 2: Establish the discipline

- Add the lint rule that flags new inline prompt strings in production code (warn at first, error after a grace period).
- Add the CI gates (artifact schema validation, eval-pass requirement).
- Document the workflow.

### 7.4 Sprint 3+: Migrate remaining prompts

In priority order (most-changed first). Each migration is one PR. The cumulative effect over 2–4 sprints is that all production prompts are now artifacts.

### 7.5 Sprint N: Move to stage 2 (versioning)

Once all prompts are artifacts, add explicit versioning per [prompt-versioning.md](./prompt-versioning.md). This is the second-stage refactor; it builds on the artifact pattern.

### 7.6 Sprint M: Move to stage 3 (runtime store)

When prompt-deploy frequency outpaces code-deploy frequency, move to a runtime store. This is the third-stage refactor and is rarely urgent — many teams stay at stage 2 forever and that is fine.

---

## 8. Integration with the LLM-call wrapper

The LLM-call wrapper described in [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) requires `prompt_version` as a parameter. The prompt-store integration makes this clean:

```python
# Application code
prompt_artifact = prompt_store.get("care_coordinator_supervisor", version="2.4.1")
messages = prompt_assembler.assemble(
    prompt_artifact,
    variables={"tenant_id": tenant_id, "user_role": role,
               "recent_history": history_summary},
)

response = llm_client.call(
    provider="anthropic",
    model="claude-opus-4-7",
    messages=messages,
    prompt_version=prompt_artifact.version_string,   # "care_coordinator_supervisor@2.4.1"
    context=context,
    ...,
)
```

The wrapper records the prompt version on the span; the trace can always reconstruct which prompt was used; the eval pipeline can correlate quality with prompt version; the cost-attribution dashboard can attribute spend per prompt version.

This integration is the *point* of the discipline. Without versioned prompts in the store, the wrapper has no version to record; without the wrapper recording, the trace cannot reconstruct.

---

## 9. Worked Meridian Health example

### 9.1 The prompt store

`meridian.prompts` is a hybrid Git + database store. Authoring is in a Git repository (`meridian-prompts/`) with PR review. A sync pipeline runs on merge to main, writing the new prompt versions to a Postgres `prompts` table. The application reads from Postgres at runtime (cached in-process with a 60-second TTL).

The Git repository has structure:

```
meridian-prompts/
├── care_coordinator/
│   ├── supervisor/
│   │   ├── v2.4.1.yaml
│   │   ├── v2.4.0.yaml
│   │   ├── v2.3.0.yaml
│   │   └── ...
│   ├── clinical_knowledge/
│   ├── drafting/
│   ├── classifier/
│   └── query_rewriter/
├── analytics_copilot/
│   ├── schema_retrieval/
│   └── sql_generation/
└── patient_api_assist/
    └── answer/
```

Each prompt is a YAML file matching the structure in §4.1.

### 9.2 The supervisor's PR workflow

When ai-platform-eng wants to ship a new supervisor version (e.g., adding a clarifying-question pattern):

1. Author creates `v2.4.2.yaml` from `v2.4.1.yaml`; edits the content.
2. PR opens. CI runs:
   - Lint: passes.
   - Fast eval: 30-case stratified sample against the clinical golden set, conversational subset, and side-effect HITL subset. Runs in 7 minutes. Pass-rate: 94% (baseline was 93%). Pass.
   - Cost regression check: estimated per-call cost rose from $0.058 to $0.061 (5%). Pass.
3. Owner-team reviewer (ai-platform-eng) approves. Because supervisor changes affect clinical interactions, a clinical-knowledge-engineering reviewer also approves.
4. Merge. Sync pipeline writes `v2.4.2` to the prompts table.
5. The application's routing layer picks up the new version on its next cache refresh (within 60 seconds).
6. Canary rollout per the platform's standard pattern: 5% traffic on the new version for 4 hours, monitor judge-pass-rate; if green, ramp to 100%; if regression, automatic rollback (the routing layer points back at `v2.4.1`).

### 9.3 The hotfix flow

In 2026-04-08, a regression was identified: a recent supervisor change had introduced a verbose intermediate-reasoning pattern that pushed average input tokens significantly higher (the cost incident referenced in [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)). The hotfix:

1. ai-platform-eng prepared a `v2.4.0-hotfix-1.yaml` reverting the verbose reasoning block.
2. PR opened with the override label (`[eval-override: reverting cost regression]`).
3. Eval-gate override approved by team lead.
4. CI passed (cost regression now reversed); reviewer approved.
5. Merged. Sync pipeline wrote the hotfix version. Routing layer was updated to point at the hotfix version. Cost normalized within the next hour.
6. The follow-up: a proper `v2.4.1.yaml` was prepared incorporating the hotfix changes with the verbose-reasoning block redesigned to not bloat tokens; eval-validated; deployed through the normal release path two days later.

### 9.4 The lint rule

A pre-commit hook (and a CI check) scans for production-code patterns that look like inline prompts:

```python
# Patterns flagged as warnings (PR comments)
SYSTEM_PROMPT_PATTERNS = [
    r'system_prompt\s*=\s*[f"\'][^"\']{50,}',
    r'system\s*=\s*[f"\'][^"\']{50,}',
    r'messages\s*=\s*\[\s*\{\s*[\'\"]role[\'\"]:\s*[\'\"]system[\'\"]\s*,\s*[\'\"]content[\'\"]:\s*[f"\'][^"\']{50,}',
]
```

The regex is heuristic; false positives are reviewed manually. The rule has caught about a dozen attempted inline-prompt additions since adoption.

### 9.5 The orphan-prompt review

Quarterly: a script scans the prompts table. Flags:
- Prompts whose owner is no longer on the platform team's roster.
- Prompts whose latest version is > 9 months old.
- Prompts whose eval suite has not been updated in > 9 months.
- Prompts not invoked in production in > 60 days.

The team reviews; either re-assigns / refreshes / retires. The 2025-Q4 review retired two prompts that were artifacts of a pilot feature; one prompt was re-assigned when the original owner left the team.

### 9.6 The platform discipline

- `meridian-prompts` is the source of truth. Application code that hardcodes prompts is rejected at code review.
- Every prompt has an owner. PR review by the owner team is required.
- Eval-gate is required for every prompt PR.
- The prompt version is on every LLM-call span (per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md)).
- Quarterly orphan-prompt review.

---

## 10. Anti-patterns

### 10.1 "We'll refactor prompts later"

The team's prompts are inline strings; the team agrees the refactor is needed but never prioritizes it. The cost of the refactor grows with every quarter of accumulation.

**Corrective.** Schedule the migration. The minimum-viable refactor (pull prompts into artifact files) is one sprint per ~20 prompts. The discipline pays back from day one.

### 10.2 "Owner is the whole team"

Every prompt's owner is "the AI team." In practice, no one is accountable; reviews happen sporadically; orphan detection is impossible.

**Corrective.** Specific team or specific person. Accountability is real or it does not exist.

### 10.3 "Prompt changes ship without eval"

A new prompt version lands; quality regressions emerge in production; the team learns the prompt change broke things by reading user complaints.

**Corrective.** Eval-gated CI is non-negotiable. Without it, prompt changes are tantamount to deploying untested code.

### 10.4 "Eval pass-rate is the only gate"

A prompt change passes the eval gate but triples per-call cost. The team finds out at the next monthly invoice review.

**Corrective.** Cost regression check is a gate alongside the eval gate. Both pass or the PR does not merge.

### 10.5 "Prompt artifact has no metadata"

The artifact is just the content string in a file. No owner, no description, no variable declarations, no depends_on. The artifact pattern provides no discipline beyond "the string lives somewhere named."

**Corrective.** The structured artifact (§4.1). The metadata is what makes the discipline real.

### 10.6 "Inline strings allowed for 'just this one quick thing'"

A particular code path was given a one-time exception to inline a prompt. The exception persisted. Other engineers see the inline prompt and follow the pattern.

**Corrective.** No exceptions. The lint rule applies uniformly. "Quick" prompts have artifacts; "test" prompts have artifacts; "experimental" prompts have artifacts.

### 10.7 "Prompt version drift between code and store"

The application code references prompt version 2.4.1; the store has 2.4.1; but the prompt-assembly logic in the application diverged from what the artifact expects (variable name changed, depends_on tool removed). Calls fail at runtime.

**Corrective.** Artifact schema validation at PR time catches mismatch. Application code that does not conform to the artifact's variable schema does not pass CI.

### 10.8 "Vendor lock-in to a prompt-management platform"

The team adopted a vendor's prompt management; prompts are stored in the vendor's proprietary format; switching vendors means re-authoring every prompt.

**Corrective.** If using a vendor, ensure the artifact format is exportable to your own neutral format (YAML, JSON). The store is replaceable; the discipline is what matters.

---

## 11. Findings (sprint-assignable)

### PROMPT-CODE-001 — Severity: Critical
**Finding.** Production prompts live as inline strings in application code; no versioning, no eval gate, no rollback path.
**Recommendation.** Begin the migration to artifact files per section 7; pull the most-changed prompt first.
**Owner.** ai-platform-eng, sprint N+1.

### PROMPT-CODE-002 — Severity: Critical
**Finding.** Prompts have no owner; quality issues do not route to a responsible team.
**Recommendation.** Assign owner per prompt; record in artifact; CI checks for valid owner.
**Owner.** ai-platform-eng team lead, sprint N+1.

### PROMPT-CODE-003 — Severity: Critical
**Finding.** Prompt changes do not go through eval-gated CI; quality regressions ship.
**Recommendation.** Eval gate on every prompt PR per section 6.3; integrate with [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md).
**Owner.** ai-platform-eng, sprint N+1.

### PROMPT-CODE-004 — Severity: High
**Finding.** Prompt artifacts have no structured metadata; downstream tooling cannot reason about owner, variables, dependencies.
**Recommendation.** Adopt the artifact structure per section 4.1; CI lint validates.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-CODE-005 — Severity: High
**Finding.** Lint rule against inline prompt strings does not exist; new prompts can still be added inline.
**Recommendation.** Add the lint rule per section 9.4; warn initially, error after a grace period.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-CODE-006 — Severity: High
**Finding.** Variable substitution is performed by application code without declaration of expected variables in the artifact.
**Recommendation.** Declare variables in the artifact; runtime substitution rejects undeclared variables; CI catches mismatches.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-CODE-007 — Severity: High
**Finding.** Cost regression check is not part of the prompt PR gate; cost-significant prompt changes ship without review.
**Recommendation.** Cost regression check per section 6.1; >20% per-call cost increase requires sign-off.
**Owner.** ai-platform-eng + finops, sprint N+3.

### PROMPT-CODE-008 — Severity: High
**Finding.** Prompts depend on tools and retrievers that are not declared; tool / retriever changes break prompts without warning.
**Recommendation.** Declare depends_on in the artifact; notify prompt owner on dependency changes; consider integration tests.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-CODE-009 — Severity: Medium
**Finding.** No orphan-prompt review; deprecated and unused prompts accumulate.
**Recommendation.** Quarterly review per section 9.5; retire or re-assign as needed.
**Owner.** ai-platform-eng team lead, sprint N+3.

### PROMPT-CODE-010 — Severity: Medium
**Finding.** Prompt review is done by general code reviewers without prompt-specific skills.
**Recommendation.** Establish prompt-reviewer track; for high-stakes prompts, include domain experts.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-CODE-011 — Severity: Medium
**Finding.** Prompt deploys are coupled to code deploys; small prompt fixes require full release cycles.
**Recommendation.** Move to runtime store (stage 3 per section 2.4); decouple prompt deploys.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-CODE-012 — Severity: Medium
**Finding.** Cross-team prompt dependencies are undocumented; changes to base prompts surprise extending-prompt owners.
**Recommendation.** Document cross-team dependencies in the artifact's depends_on.prompts; notify on base changes.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-CODE-013 — Severity: Medium
**Finding.** Hotfix flow is undocumented; urgent prompt fixes follow ad-hoc patterns.
**Recommendation.** Document the hotfix workflow per section 6.2; train the team.
**Owner.** ai-platform-eng + sre, sprint N+4.

### PROMPT-CODE-014 — Severity: Medium
**Finding.** Prompt-store access controls are absent; any engineer can update any prompt.
**Recommendation.** Per-team or per-prompt access controls; integration with the existing identity model.
**Owner.** ai-platform-eng + security-eng, sprint N+4.

### PROMPT-CODE-015 — Severity: Low
**Finding.** Few-shot examples are embedded in the prompt content rather than declared as separate examples.
**Recommendation.** Separate examples field in the artifact; CI validates example shape; supports per-example A/B testing.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-CODE-016 — Severity: Low
**Finding.** Prompt deprecation lifecycle is implicit; old versions are removed without notice.
**Recommendation.** Formalize per [prompt-versioning.md](./prompt-versioning.md); deprecation notice; minimum-grace-period before removal.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-CODE-017 — Severity: Low
**Finding.** Prompt store is unable to answer "which prompts use model X" or "which prompts depend on tool Y."
**Recommendation.** Index the depends_on metadata; build query interface; surface in dashboards.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-CODE-018 — Severity: Low
**Finding.** No platform-level reporting on prompt-deploy frequency, prompt PR cycle time, or prompt-review latency.
**Recommendation.** Add platform-health metrics for prompt operations.
**Owner.** ai-platform-eng team lead, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team starting from "our prompts are inline strings":

- [ ] **Sprint 0 — inventory.** Catalog production prompts. Identify owners de facto. Pick the most-changed for the first migration.
- [ ] **Sprint 1 — first migration.** Pull the first prompt into an artifact file. Verify behavior unchanged. Land.
- [ ] **Sprint 1 — store choice.** Decide between Git, database, vendor, or hybrid per section 3.
- [ ] **Sprint 2 — discipline.** Lint rule against inline prompts. CI gates (schema lint, eval pass).
- [ ] **Sprint 2 — owner assignment.** Every prompt has a named owner. CI checks.
- [ ] **Sprint 3 — migrate the rest.** Remaining prompts to artifacts. The longer the team waits, the harder this gets.
- [ ] **Sprint 3 — eval-gate enforcement.** Eval pass required on every prompt PR. Override pattern documented.
- [ ] **Sprint 4 — cost regression gate.** Cost check on prompt PRs.
- [ ] **Sprint 4 — depends_on declaration.** Variables, tools, retrievers, other prompts.
- [ ] **Sprint 5 — versioning.** Add explicit versions per [prompt-versioning.md](./prompt-versioning.md).
- [ ] **Sprint 6+ — runtime store (if needed).** When prompt-deploy frequency outpaces code-deploy frequency, move to stage 3.
- [ ] **Ongoing — quarterly review.** Orphan prompts, owner currency, eval coverage.

A team that completes this sequence has the prompt-engineering discipline that makes everything else (eval gates, observability with prompt versions, canary rollouts, rollback) possible. A team that defers carries the inline-strings tax forever and pays it in production incidents.

---

## 13. References

- The "configuration as code" pattern from the cloud engineering canon — the prompts-as-code pattern is the same shape applied to a different artifact.
- LangSmith Prompts, Braintrust Prompts, Helicone Prompts — vendor prompt management options.
- Git, DVC, MLflow — open-source patterns for versioned artifact storage that apply.
- This repo: [prompt-engineering/prompt-versioning.md](./prompt-versioning.md) — the companion document on version lifecycle.
- This repo: [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — the eval gate that prompt PRs depend on.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — the wrapper that records the prompt version on every call.
- This repo: [cicd-and-eval-gates/prompt-version-pinning.md](../cicd-and-eval-gates/) (coming) — the deployment-side pinning practice.
- Sibling repo: [ai-architecture-reference-architecture/context-and-prompt-architecture/](https://github.com/jeremiahredden/ai-architecture-reference-architecture/tree/main/context-and-prompt-architecture) — the architectural choices about prompt assembly.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the architecture context. ARCH-CARE-005 is the cross-link finding.
