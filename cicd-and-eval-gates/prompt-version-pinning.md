# Prompt Version Pinning

> **Audience.** Engineers refactoring a system where prompts are edited in production through admin UIs or notebooks. Platform leads who have asked "which prompt was running last Tuesday?" and not gotten an answer. Anyone whose production incident traced back to a prompt change made without a PR. **Scope.** The *engineering* pattern for pinning prompt versions in release artifacts and enforcing the pin at deploy time: hash-based prompt artifacts, prompt-store-as-dependency, deployment integration that fails on missing pins, rollback that includes prompts alongside code. Pair with [model-version-pinning.md](./model-version-pinning.md) (the model-side companion) and [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) (the pipeline that enforces the pin). Cross-link to [prompt-engineering/prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md) (the prompt-side discipline this pin enforces) and [prompt-engineering/prompt-versioning.md](../prompt-engineering/prompt-versioning.md) (the version-numbering convention this pin references). **Worked client.** Meridian Health.

---

## 1. Why this document exists

A prompt is a load-bearing artifact. One sentence in the system prompt can shift quality across every conversation the feature serves. Format compliance, refusal behavior, tone, tool-call selection, factuality — all of these are influenced by the prompt as much as by the model. Yet on most teams, the prompt is treated as runtime configuration: editable on the fly, often through an admin UI, often without version control, almost never pinned at release time.

The discipline gap is the same one model version pinning closes ([model-version-pinning.md](./model-version-pinning.md)) but for the prompt. Without pinning, the prompt in production is a *runtime* property: whatever the prompt store happens to be serving at the moment of the call. With pinning, the prompt in production is a *release-time* property: explicitly committed in the release artifact, validated at deploy, and visible in every trace.

The Care Coordinator's findings around prompt drift (a long line of `ARCH-CARE-*` findings from prior batches) are the symptom. The pattern this document describes is the structural fix.

The honest framing: prompts that are not pinned are not under change control. Production behavior can shift without a PR, without an eval, without an approval, without a rollback path. The team is one careless admin-UI edit away from a quality regression that no review caught. Pinning is what closes that gap.

This document is opinionated about four things:

1. **Every prompt in the release manifest is pinned to a content hash.** Not a label, not a "production" tag. The exact bytes of the prompt are committed.
2. **The prompt store is a dependency, not a database.** The release pulls prompts as part of build, not at runtime.
3. **Deploy gate enforces the pin.** A release that pins a non-existent prompt version is refused at deploy time.
4. **Rollback restores prompts alongside code.** A code rollback to release N also restores the prompt versions release N was pinned to.

Structure: (2) what "pinning" means for a prompt; (3) the release manifest; (4) the prompt store as dependency; (5) the deploy gate; (6) the build-time bundling; (7) integration with the prompt-versioning discipline; (8) rollback discipline; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. What "pinning" means for a prompt

The pin is a release-time commitment to a specific prompt artifact.

### 2.1 Specifics of the pin

A prompt pin specifies:

- **The prompt's logical name.** `care_coordinator_supervisor`, not "supervisor v3."
- **The version.** Semantic version (`2.4.1`) or content hash (`sha256:7f9a...`). Both have uses.
- **The hash, always.** Even when the version is semantic, the hash is recorded. A semantic version is a label; the hash is what was actually shipped.
- **The variables it expects.** A prompt that expects `{user_input}` and `{retrieved_docs}` declares this in its manifest; the deploy gate verifies the calling code provides them.

### 2.2 Hash-based vs version-based

A *content hash* is the SHA256 (or similar) of the prompt's exact text. A *version* is a human-readable label assigned to a content hash.

Pros of hash:

- Cryptographically tamper-evident. Two prompts with the same hash are byte-for-byte identical.
- Reproducible across systems. Two teams can verify they shipped the same prompt by comparing hashes.
- Catches accidental edits. A prompt that has been silently modified produces a different hash.

Pros of version:

- Human-readable.
- Can carry semantic meaning (`2.4.1` implies compatibility with `2.4.0` callers).
- Easier in changelogs and conversations.

The pattern: use both. The version is the human-readable identifier; the hash is the cryptographic guarantee. The release manifest pins both.

### 2.3 What pinning prevents

- **Silent prompt edits.** Someone changes the prompt in the admin UI; the next deploy reads a different prompt; behavior shifts without anyone knowing.
- **Environment drift.** Dev points at `latest`, prod points at `latest`, and they end up running different prompt versions.
- **Reproducibility loss.** "What prompt was running last Tuesday?" becomes unanswerable without the pin.
- **Audit gaps.** Compliance reviewers cannot attest to which prompt was used for a regulated decision.
- **Rollback failure.** Rolling back code without rolling back prompts leaves the system in an inconsistent state.

### 2.4 What pinning does NOT prevent

- **Bad prompts shipping.** Pinning is a contract, not a quality check. The eval gate is what catches bad prompts ([eval-gate-design.md](./eval-gate-design.md)).
- **Prompt drift over time.** Pinning preserves the prompt; humans still drift it by writing new versions. Versioning discipline ([prompt-versioning.md](../prompt-engineering/prompt-versioning.md)) is what manages drift.
- **Behavior changes from model upgrades.** A pinned prompt against a different model version may behave differently. The model pin is the companion.

---

## 3. The release manifest

The release manifest specifies the pinned set for a release.

### 3.1 Manifest shape

```yaml
release:
  version: 2026.05.25-r3
  code_sha: 9c2a1f8b...
  timestamp: 2026-05-25T14:30:00Z

  prompts:
    care_coordinator_supervisor:
      version: 2.4.1
      sha256: 7f9a3b2c8d1e...
      expects_variables: [user_input, conversation_history, patient_context, retrieved_clinical_facts]
      schema_version: 1.2.0

    care_coordinator_classifier:
      version: 1.2.0
      sha256: 3c8e2a1f4b9d...
      expects_variables: [user_input]
      schema_version: 1.0.0

    care_coordinator_clinical_knowledge:
      version: 3.1.0
      sha256: 9d2a8c4f1b7e...
      expects_variables: [query, retrieved_docs]
      schema_version: 2.1.0

    care_coordinator_drafting:
      version: 2.0.4
      sha256: 1b8f7e3a2c9d...
      expects_variables: [intent, context, tone]
      schema_version: 1.3.0

    # ... per prompt

  models:
    # ... (see model-version-pinning.md)

  datasets:
    # ... (see dataset-version-pinning.md, coming)

  eval_suite:
    version: 4.2.0
    sha256: ...

  signed_by: ci-bot
  approved_by:
    - ai-platform-lead
    - product-owner
```

### 3.2 What the manifest enforces

A release that omits a prompt the application code uses is refused at deploy time. Both directions checked:

- For every prompt name the application code requests at runtime, the manifest must have a pin.
- For every prompt the manifest pins, the prompt store must have the artifact at that hash.

This is symmetric: missing pins fail, extra pins are warnings (probably a removed code path that should have removed the pin).

### 3.3 Schema-versioned prompts

A prompt has a *schema* — the set of variables it expects to receive. The schema is versioned independently of the prompt content. A prompt update that adds a new variable bumps the schema version (breaking change for callers); a prompt update that just refines wording does not (compatible change).

The deploy gate verifies that the calling code is at a schema version compatible with the pinned prompt's schema version.

### 3.4 Multi-prompt releases

A release that updates multiple prompts is normal. The manifest carries all of them. The eval gate runs against the full set (not prompt-by-prompt) so cross-prompt interactions are caught.

---

## 4. The prompt store as dependency

The prompt store is the system that holds versioned prompt artifacts. It is the *dependency*, not the runtime.

### 4.1 Why it's a dependency

A dependency is fetched at build time, locked at release time, and immutable in deployment. A runtime config is fetched at runtime and can change while the app is running. Prompts should be the former.

This is the conceptual flip from how many teams treat prompts. The team's instinct is to say "prompts should be hot-reloadable; we want to change them without redeploying." That instinct produces the problem: hot-reloadable prompts are unpinnable, untraceable, and unauditable.

### 4.2 What "fetched at build time" looks like

The release-build pipeline:

1. Reads the prompt manifest.
2. For each pinned prompt: fetches the prompt content from the store at the pinned hash.
3. Verifies the fetched content's hash matches the manifest.
4. Bundles the prompts into the release artifact (either as files in the container image or as a release-attached package).
5. The deployed runtime reads prompts from the bundled artifact, not from the store.

After deploy, the prompt store can be edited freely; the running release continues to use its bundled prompts. The next release pulls again from the store.

### 4.3 The prompt store implementation

Options:

- **Git repository.** Each prompt is a file; versions are git commits; hashes are git object hashes. Simple, audited, low-cost. Default choice for most teams.
- **Specialized prompt-management product.** PromptLayer, Helicone, internal-platform tooling. Adds workflow features (review, A/B, analytics); pulls the prompt as part of build remains the discipline.
- **Document store.** S3 + version metadata, DynamoDB + content hash, etc. Custom but workable.

The store *is not the runtime*. The runtime is the bundle inside the release.

### 4.4 Edit workflow

When the prompt store is git:

- Engineer edits the prompt in a feature branch.
- PR opens with the diff.
- Fast eval runs on the candidate prompt against the rest of the production set.
- Reviewer approves.
- PR merges; the prompt store has the new version.
- The next release-build run includes the new version *if* the release manifest is updated to pin to it.
- A release is opened to bump the manifest's pin for that prompt; the release goes through the standard pipeline.

The PR to the prompt store is not itself a release. The release happens when the manifest is updated and the standard pipeline runs.

### 4.5 Hot-reload? When?

Some workflows want hot-reload for development convenience. The discipline:

- Dev / staging environments: hot-reload allowed.
- Production: never. Production reads from the bundled artifact in the release.

If production "needs" hot-reload, the actual need is faster release cadence, not bypassing the release.

---

## 5. The deploy gate

The deploy gate enforces the pin. It is the last line of defense against an unpinned or mismatched prompt reaching production.

### 5.1 What the deploy gate checks

Before a release deploys:

1. **Every pin resolves.** For every prompt named in the manifest, the prompt store has an artifact at the specified hash.
2. **Every used prompt is pinned.** For every prompt name the deploying code requests, the manifest has a pin. (Discovered by static analysis of the code's prompt-load calls.)
3. **Schema compatibility.** The code's prompt-schema-version requirements are satisfied by the pinned prompts' schema versions.
4. **No deprecated prompts.** Prompts marked deprecated in the prompt-store metadata cannot be pinned in new releases.
5. **No `latest` references.** Pins are to specific hashes, never to `latest` or similar aliases.

If any check fails, the deploy refuses.

### 5.2 Where the gate lives

The gate is a CI step that runs after the release manifest is built and before the deployment is created.

```yaml
deploy_gate:
  runs:
    - validate_release_manifest_schema
    - check_all_prompts_resolve_in_store
    - check_no_alias_references
    - check_no_deprecated_prompts
    - validate_prompt_schema_compatibility
    - check_against_code_static_analysis
  on_failure: block_deploy
```

### 5.3 The failure modes

When the gate blocks, the failure message is specific:

- "Prompt `care_coordinator_supervisor@2.4.1@sha256:7f9a...` not found in store."
- "Code references prompt `care_coordinator_intent_detector` but manifest has no pin."
- "Pin `care_coordinator_classifier@2.0.0` requires schema 2.x but caller code expects schema 1.x."
- "Prompt `legacy_summarizer@1.0.0` is deprecated and cannot ship in new releases."

The engineer sees the exact problem; the fix is direct.

### 5.4 Bypass discipline

The deploy gate has the same override mechanics as the eval gate:

- Documented justification.
- Senior approval (SRE on-call + AI Platform lead).
- Logged in the release artifact.
- Expiration; lapsed overrides re-block.

The bypass exists for emergencies. Not for convenience.

---

## 6. Build-time bundling

How the prompts get into the release.

### 6.1 The bundling step

Post-pin-validation:

1. The build pulls each pinned prompt from the store at its pinned hash.
2. The build verifies each fetched prompt's hash matches the manifest.
3. The build copies the prompts into the release artifact:
   - For container deploys: copied into the image at a known path (`/app/prompts/`).
   - For serverless deploys: bundled with the deployment package.
   - For VM deploys: bundled with the artifact tarball.
4. The release artifact records the bundled set; the deployed runtime reads from the bundle.

### 6.2 The runtime's prompt-loading discipline

The runtime loads prompts from the bundled location:

```python
class PromptLoader:
    def __init__(self, bundle_path: str = "/app/prompts/"):
        self.bundle_path = bundle_path
        self.prompts = self._load_bundle()

    def _load_bundle(self) -> dict:
        # Loads every prompt file from the bundle on startup.
        # Verifies each file's hash matches the manifest record.
        # Stores them in memory.

    def get(self, name: str) -> str:
        if name not in self.prompts:
            raise PromptNotPinned(name)
        return self.prompts[name]
```

A `PromptNotPinned` exception at runtime is a build-time failure that escaped (should be impossible if the deploy gate is wired correctly).

### 6.3 No runtime fetch from store

The runtime *does not* call out to the prompt store. The bundle is everything. This matters for:

- **Reliability.** A prompt-store outage cannot affect a running deployment.
- **Performance.** No network call for prompt loading on the hot path.
- **Security.** The prompt-store doesn't need network reachability from production runtime.
- **Reproducibility.** A given release loaded a given set of prompt bytes; this is not subject to "the store happened to be in a different state at runtime."

### 6.4 Multi-region / multi-tenant nuances

For per-tenant prompts ([multi-tenancy-and-isolation/per-tenant-prompt-and-context.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-prompt-and-context.md)): the per-tenant overrides are bundled too. The manifest pins them per-tenant:

```yaml
prompts:
  care_coordinator_supervisor:
    default:
      version: 2.4.1
      sha256: 7f9a3b...
    tenants:
      hospital_a:
        version: 2.4.1-hospital_a
        sha256: 4d2c8b...
      hospital_b:
        version: 2.4.1-hospital_b
        sha256: 8e1a9f...
```

All tenant variants are pinned and bundled. The runtime routing layer selects the appropriate variant based on tenant.

---

## 7. Integration with the prompt-versioning discipline

The pinning pattern works only if the upstream prompt-versioning discipline is sound.

### 7.1 The dependency on prompt-versioning.md

[prompt-engineering/prompt-versioning.md](../prompt-engineering/prompt-versioning.md) defines:

- The semantic-versioning scheme (major / minor / patch).
- The change-classification convention (breaking change → major bump; new functionality → minor; tweak → patch).
- The "what counts as a change" rules.
- The deprecation policy.

Without that discipline, version numbers are arbitrary and pinning is just hash-pinning with a label.

### 7.2 What this layer adds

This document adds the *release-artifact integration*: how the versioned prompts get into the release, get bundled, get enforced at deploy.

### 7.3 The CI integration with versioning

When a PR edits a prompt:

- Lint checks the version bump matches the change type (a breaking change without a major bump fails lint).
- Fast eval catches obvious regressions.
- The PR merges; the prompt store has the new version.
- A separate "release prep" PR bumps the release manifest's pin to the new version.
- The release goes through the pipeline.

The two-PR pattern (one for the prompt change, one for the release pin) is the discipline. It ensures every release is an explicit decision; promoted versions are intentional.

### 7.4 Skipping the two-PR pattern

In practice, low-risk prompt changes can be bundled with the manifest pin in one PR. The two-PR pattern is mandatory for:

- Breaking changes (schema version bump).
- Critical-case-affecting changes.
- Per-tenant prompts being pushed to additional tenants.

Low-risk patch changes (wording tweaks, format clarifications) can be one-PR for velocity.

---

## 8. Rollback discipline

Rollback is where pinning earns its keep.

### 8.1 The full-system rollback

Per [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md), a code rollback restores not just code but the *full pinned set* — the prior release's prompts, models, datasets, and eval suite.

The mechanism:

- The prior release's manifest is preserved (retention window: typically 14 days).
- Rollback flips the deployment's release pointer to the prior release.
- The prior release's bundle is what gets deployed; the prior prompts are restored as a side effect of restoring the prior release.

### 8.2 Why prompts must roll back with code

A common failure mode: rolling back code without rolling back prompts. Result: old code with new prompts, or new code with old prompts. Both are *unsupported configurations* — they were not what was eval'd, were not what was canary'd, and may behave in ways no one expects.

The discipline: full pinned-set rollback is the only rollback. Partial rollback (e.g., "roll back only the prompts") is a different operation: it requires its own release, with its own eval and canary.

### 8.3 Prompt-only rollback as a release

When the prompt is the problem (not the code):

- The team identifies the prior known-good prompt version.
- A new release is opened that pins to the prior prompt version with current code.
- The release goes through the standard pipeline (eval, canary).
- The release deploys.

This is not "rollback" — it is "forward-fix with a prior prompt." The conceptual distinction matters: it goes through full change control rather than emergency restoration.

### 8.4 Emergency prompt-only restoration

For genuine emergencies (a safety incident traceable to a prompt change), the emergency-hotfix path ([pipeline-architecture-for-ai.md §9.5](./pipeline-architecture-for-ai.md)) applies:

- An emergency hotfix release pins the prior known-good prompt.
- The release goes through abbreviated eval (lint + fast eval; full eval in background).
- Canary at 1% for 15 minutes (shortened window).
- Promote.

Emergency restoration is allowed because the alternative is letting the incident continue. The retro reviews whether the standard process should have caught the issue.

---

## 9. Worked Meridian example: refactoring the Care Coordinator prompts

The Care Coordinator team has 12 prompts in production: supervisor, classifier, intent-detector, clinical-knowledge, drafting, tone-adjuster, escalation-handler, refusal-handler, query-rewriter, summary-generator, follow-up-suggester, handoff-formatter. Eight of them have version `*.4.x` or older. The team wants to refactor them all to a unified style.

### 9.1 The refactor approach

- 12 PRs to the prompt store, one per prompt. Each PR bumps to a new minor version.
- Each PR runs fast eval; reviewer approval per prompt.
- Once all 12 are merged to the prompt store: a single release-prep PR bumps the manifest pins.
- The release goes through full pipeline.

### 9.2 The release-prep PR

The PR changes the release manifest:

```diff
prompts:
  care_coordinator_supervisor:
-    version: 2.4.1
-    sha256: 7f9a3b2c8d1e...
+    version: 2.5.0
+    sha256: 9d3a8c4f1b7e...

  care_coordinator_classifier:
-    version: 1.2.0
-    sha256: 3c8e2a1f4b9d...
+    version: 1.3.0
+    sha256: 4b9d3c8e2a1f...

  # ... 10 more prompts ...
```

### 9.3 The pipeline

- Lint passes.
- Fast eval runs on 150 cases against all 12 updated prompts: catches no critical-case failures, pass rate 99.0% vs baseline 99.2%. Within tolerance.
- PR reviewed by AI Platform lead + clinical content reviewer; both approve.
- Merge; release candidate built.
- Full eval runs against all 12 prompts; 1247 cases; pass rate 99.38% vs baseline 99.42%. Within tolerance.
- Cost: +1.1%. Latency: -0.4%. Within tolerance.
- Cross-feature regression: no new failures.
- Canary at 1% for 4 hours; live-judge clean. Auto-ramp.
- 10% canary 4 hours. 50% canary 12 hours, human-gate. Approve. 100%.

### 9.4 Three weeks later: rollback drill

The SRE team runs a quarterly rollback drill. They roll back the production deployment from release `2026.05.25-r3` to `2026.05.18-r2` (the prior known-good).

The rollback:

- Flips the deployment pointer to the `2026.05.18-r2` manifest.
- The runtime reads prompts from the `2026.05.18-r2` bundle.
- All 12 prompts revert to their prior versions.
- Models revert to their prior pins (same in this case; the model pins did not change).
- Code reverts to the prior commit.

Verification: a tail of production traces shows the prompt-version metadata in every span. Every span now references the prior version. The rollback is clean.

### 9.5 The audit trail

A compliance reviewer asks: "Which prompt was running for tenant H on 2026-05-22 at 14:30 UTC?"

The answer is recoverable:

- The deployment log shows release `2026.05.25-r3` was live at that time.
- The release manifest for `2026.05.25-r3` shows the tenant-H supervisor pin: `2.5.0-hospital_h@sha256:8e1a9f...`.
- The prompt store at that hash returns the exact bytes that were running.
- The eval result for that release artifact shows the quality metrics at that time.

The audit trail is a function of the pinning discipline.

### 9.6 Findings closed

- **ARCH-CARE-069** (prompts edited via admin UI; no PR record, no eval, no rollback).
- **ARCH-CARE-070** (prompt store and release artifact disconnected; "which prompt" was unanswerable).
- **ARCH-CARE-071** (rollback rolled back code without rolling back prompts; ran in unsupported config).
- **ARCH-CARE-072** (no deploy gate for prompts; missing pins reached production).
- **ARCH-CARE-073** (per-tenant prompts not pinned per-tenant; tenant overrides shipped without traceability).

---

## 10. Anti-patterns

### 10.1 The hot-reload-from-store production

The runtime reads prompts from the store on every call (or on a short-TTL cache refresh). An admin edits the store; production behavior shifts immediately. Quality regressions ship without any release. The team has no answer to "what was running when."

The fix: prompts are bundled into the release at build time; runtime never reads from the store.

### 10.2 The alias pin

The manifest pins to `latest` or `production` or `v2`. The actual artifact those aliases resolve to can change without the manifest changing. The pin is a lie.

The fix: pin to specific hashes. Lint rule rejects alias pins.

### 10.3 The forgotten manifest

The team updates prompts in the store but forgets the release-prep PR. The next deploy uses the *prior* prompts, not the new ones. The team is confused why their "update" did not take effect.

The fix: the prompt store and the release manifest are *both* updated in any prompt change; the two-PR or one-PR pattern enforces this.

### 10.4 The deploy without the gate

The deploy gate exists but is not in the deploy path (e.g., it's a CI check but not a deploy block). A release with missing pins reaches production. The runtime fails with `PromptNotPinned` on the first request that uses the missing prompt.

The fix: the deploy gate is in the deploy path. A failure blocks deploy, not just notifies.

### 10.5 The prompt-only "rollback"

A bad prompt change shipped. The team rolls back only the prompt, leaving the code from the latest release. The rollback creates a configuration that was never eval'd; new issues emerge.

The fix: full-system rollback. Prompt-only fix is a new release, not a rollback.

### 10.6 The schema-version-blind pin

The code expects prompt schema 2.x; the manifest pins a 1.x prompt. The deploy gate doesn't check schema compatibility; the runtime fails on missing variables.

The fix: schema-version check in the deploy gate.

### 10.7 The deprecated prompt re-shipped

A prompt was marked deprecated three months ago. Someone re-pins it for a quick fix; the deprecation goes unenforced.

The fix: deploy gate refuses deprecated prompts. The deprecation policy is enforced in CI, not in convention.

### 10.8 The per-tenant override-as-runtime-config

Per-tenant prompts are stored in the production database, updated by support engineers, never PR-reviewed, never pinned. The system has many "production prompts" that nobody can enumerate.

The fix: per-tenant overrides are pinned in the manifest. The store is still the dependency; the bundle still contains everything. Support engineers cannot edit production directly.

---

## 11. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| CICD-PVP-001 | Prompts read from store at runtime; production hot-reload in effect | High | Bundle prompts in release artifact; runtime reads bundle only | AI Platform + SRE |
| CICD-PVP-002 | Manifest pins use aliases (`latest`, `production`); pin not deterministic | High | Pin to specific content hashes; lint rule rejects aliases | AI Platform |
| CICD-PVP-003 | Prompt edits skip PR / review; admin-UI edits reach production | High | Branch-protect prompt store; admin UI removed or restricted to dev environments | AI Platform + SRE |
| CICD-PVP-004 | Deploy gate absent or non-blocking; missing pins reach production | High | Deploy gate per §5; failures block deploy | AI Platform + SRE |
| CICD-PVP-005 | Schema-version compatibility unchecked; runtime fails on variable mismatch | High | Schema-version check in deploy gate per §5.1 | AI Platform |
| CICD-PVP-006 | Deprecated prompts can be re-pinned; deprecation policy unenforced | Medium | Deploy gate refuses deprecated prompts per §5.1 | AI Platform |
| CICD-PVP-007 | Rollback rolls back code only; prompts not restored | High | Full-pinned-set rollback per §8.1; verified in rollback drills | AI Platform + SRE |
| CICD-PVP-008 | Per-tenant prompts not pinned per-tenant in manifest | Medium | Per-tenant pins per §6.4; bundled with release | AI Platform + Architecture |
| CICD-PVP-009 | Audit trail incomplete; "which prompt was running" unanswerable | High | Manifest preserved per release; release-id captured in every trace | AI Platform + Observability |
| CICD-PVP-010 | Prompt store unversioned; no source-of-truth for "what was at hash X" | High | Use git or versioned object store; immutable content addressing | AI Platform |
| CICD-PVP-011 | Two-PR pattern (store update + manifest pin) skipped for breaking changes | Medium | Enforce two-PR pattern for major schema bumps per §7.4 | AI Platform |
| CICD-PVP-012 | Production runtime calls out to prompt store; store outage impacts production | High | Bundle prompts; runtime independent of store availability | AI Platform + SRE |
| CICD-PVP-013 | Prompt deprecations communicated by chat; no machine-readable status | Medium | Deprecation status in prompt-store metadata; deploy gate reads it | AI Platform |
| CICD-PVP-014 | Hash recorded only, version label absent; engineer-facing readability poor | Low | Pin both version and hash per §2.2 | AI Platform |
| CICD-PVP-015 | Bundle integrity not verified at runtime startup | Medium | Runtime verifies bundled prompt hashes against manifest at boot per §6.2 | AI Platform |
| CICD-PVP-016 | Quarterly rollback drill absent; rollback discipline untested | Medium | Quarterly rollback drill in staging; verify full-set restore | SRE |
| CICD-PVP-017 | Per-environment pin drift (dev / staging / prod pin differently) | Medium | Manifest is single source of truth; all environments pin from it | AI Platform |
| CICD-PVP-018 | Prompts shipped without eval (lint passed, fast eval skipped on prompt-only PR) | High | Fast eval triggered on `prompts/` path changes per [pipeline-architecture-for-ai.md §4.5](./pipeline-architecture-for-ai.md) | AI Platform + Eval Eng |

---

## 12. Adoption checklist

- [ ] Release manifest pins every prompt in use; manifest is the source of truth.
- [ ] Pins reference specific content hashes; no aliases (`latest`, `production`).
- [ ] Each pin includes version, hash, expected variables, schema version.
- [ ] Prompt store is the build-time dependency; runtime reads from bundled artifact only.
- [ ] Branch protection on prompt store; admin-UI direct edits to production blocked.
- [ ] Deploy gate verifies: every pin resolves, every used prompt is pinned, schema compatibility, no aliases, no deprecated prompts.
- [ ] Deploy gate failures block deploy (not just warn).
- [ ] Per-tenant prompt overrides pinned in manifest; bundled with release.
- [ ] Full-pinned-set rollback discipline; partial rollback prohibited.
- [ ] Rollback drilled quarterly in staging; full-restore verified.
- [ ] Two-PR pattern enforced for breaking changes; one-PR allowed for low-risk patches.
- [ ] Release manifest preserved per release; archived for audit duration.
- [ ] Release-id captured in every production trace; "which prompt was running" answerable.
- [ ] Runtime verifies bundled hashes against manifest at boot.
- [ ] Deprecation status machine-readable in prompt-store metadata.

---

## 13. References

**Internal:**

- [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) — the pipeline that enforces the pin.
- [eval-gate-design.md](./eval-gate-design.md) — the quality gate complementary to the pin.
- [model-version-pinning.md](./model-version-pinning.md) — the companion pin for model versions.
- [release-artifacts-for-ai.md](./) — the artifact format containing the pinned set (coming).
- [dataset-version-pinning.md](./) — the companion pin for datasets (coming).
- [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md) — the rollback discipline that restores pinned prompts.
- [model-lifecycle/canary-and-shadow-rollout.md](../model-lifecycle/canary-and-shadow-rollout.md) — the canary mechanic for releases including new prompts.
- [prompt-engineering/prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md) — the discipline this pin makes operationally real.
- [prompt-engineering/prompt-versioning.md](../prompt-engineering/prompt-versioning.md) — the versioning convention pins reference.
- [prompt-engineering/prompt-libraries-and-components.md](../prompt-engineering/prompt-libraries-and-components.md) — the modularization the version-pin pattern enables.
- [prompt-engineering/prompt-as-api-contract.md](../prompt-engineering/prompt-as-api-contract.md) — the schema discipline the deploy gate enforces.
- [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — the trace shape recording prompt version per call.

**Cross-repo (architecture sibling):**

- [context-and-prompt-architecture/system-prompt-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/context-and-prompt-architecture/system-prompt-architecture.md) — architecture-side framing of the system prompt.
- [context-and-prompt-architecture/prompt-as-api-discipline.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/context-and-prompt-architecture/prompt-as-api-discipline.md) — architecture-side framing of the prompt-as-contract pattern.
- [multi-tenancy-and-isolation/per-tenant-prompt-and-context.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-prompt-and-context.md) — per-tenant prompt overrides pattern.
- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
