# Pipeline Architecture for AI

> **Audience.** Engineers and platform leads who are responsible for shipping AI changes through CI/CD. Anyone whose team currently edits a prompt in a notebook, copies it into production, and prays. Tech leads who have asked the question: *the rest of the company ships changes through a pipeline; why does the AI team's process look like a 2012 startup?* **Scope.** The *engineering* design of a CI/CD pipeline for AI features: the stages (lint → fast eval → full eval → cost-regression → latency-regression → canary → monitor → promote); the gate criteria; the branch-protection rules; the integration with the eval engineering folder; the artifact format. Pair with [eval-gate-design.md](./eval-gate-design.md) (the load-bearing gate) and [release-artifacts-for-ai.md](./) (the artifact format, coming). Cross-link to [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) (the upstream eval design that the pipeline integrates with) and [model-lifecycle/model-promotion.md](../model-lifecycle/model-promotion.md) (the model-side workflow this pipeline serves). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The rest of the engineering org ships changes through a CI/CD pipeline. A code change opens a PR; the PR triggers tests, linting, security scans; reviewers approve; the change merges and deploys through a series of gates and environments. This discipline took the industry a decade to build and is now load-bearing infrastructure for any serious software team.

AI changes — prompt edits, model-version bumps, retrieval-corpus refreshes, fine-tune deployments — usually bypass this discipline entirely. The pattern I see most often: a prompt is edited in a notebook, the engineer eyeballs a few examples, the change is copied into the production prompt store, and traffic starts hitting the new prompt within minutes. There is no PR, no review, no eval gate, no canary, no rollback path. The blast radius of the change is every conversation the feature serves; the change-control discipline is none.

The cost of this gap is measurable: most production AI incidents in 2026 trace to a change that bypassed CI/CD discipline. A prompt edit that worked on the engineer's test cases but failed in production on a class of queries the test cases did not cover. A model-version bump applied during a notebook session that quietly changed format compliance across thousands of downstream consumers. A retrieval-corpus refresh that introduced a contaminated document the eval suite would have caught if the eval suite had been run.

The fix is not "more careful manual review." The fix is to treat AI changes as code changes and put them through the same pipeline as code — with the additions that AI changes need (eval gate, cost gate, latency gate, canary discipline) wired in. The pipeline mechanics from the rest of the org carry over almost entirely; what changes is the gate criteria and the artifact contents.

This document is opinionated about four things:

1. **Every AI change goes through CI/CD.** Prompts, models, datasets, retrieval corpora, fine-tunes. No notebook-edit-and-ship path. Branch protection enforces the rule.
2. **The eval gate is the load-bearing CI step.** Without it the rest of the pipeline is procedural decoration. The eval gate is what makes "this change passed CI" mean something for AI features.
3. **AI changes default to canary, not direct cutover.** The failure modes are hard to detect in pre-production; the canary is what catches the regressions the eval gate misses.
4. **Pipelines are AI-specific in their gates but generic in their structure.** Reuse the team's existing CI infrastructure (GitHub Actions, GitLab, Jenkins, Buildkite). Don't build a parallel AI-only system; integrate.

Structure: (2) the pipeline stages; (3) the lint stage; (4) the fast eval; (5) the full eval; (6) the cost and latency gates; (7) the canary stage; (8) the promote stage; (9) the branch protection model; (10) worked Meridian example; (11) anti-patterns; (12) findings; (13) adoption checklist; (14) references.

---

## 2. The pipeline stages

A working CI/CD pipeline for AI has the following stages, in order:

```
[ PR open ] → [ lint ] → [ fast eval ] → [ review ] → [ merge ]
              ↓
              [ full eval (nightly + release candidate) ]
              ↓
              [ cost-regression gate ]
              ↓
              [ latency-regression gate ]
              ↓
              [ canary deploy at 1% ]
              ↓
              [ monitor canary ]
              ↓
              [ ramp 10% → 50% → 100% ]
              ↓
              [ promote ]
```

The stages map to two trigger points: per-PR (lint, fast eval) and per-release (everything else).

### 2.1 Per-PR stages

Run on every push to a feature branch. Must complete fast enough that engineers do not bypass them.

- **Lint.** Static checks on the AI artifacts: prompt structure validity, schema validity, prompt-store integrity, no hard-coded secrets, no model aliases. < 30 seconds.
- **Fast eval.** A small, fast subset of the full eval suite. Catches obvious regressions before review. Target < 10 minutes.

### 2.2 Per-release-candidate stages

Run when a release candidate is built (typically post-merge to main).

- **Full eval.** The complete eval suite ([eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md)). Hours, sometimes overnight.
- **Cost-regression gate.** Verify the candidate's projected cost per request has not regressed.
- **Latency-regression gate.** Verify the candidate's p50/p95 latency has not regressed.

### 2.3 Per-deploy stages

Run when a release candidate is promoted to a deployment.

- **Canary at 1%.** New version receives 1% of traffic. Monitor against canary criteria.
- **Ramp at 10%, 50%, 100%.** Each step gated on canary criteria.
- **Promote.** Full traffic to the new version; old version retained for rollback.

### 2.4 Nightly / weekly stages

Run on a schedule, independent of PR or deploy activity.

- **Full eval on main.** Confirms main branch is in a shippable state.
- **Quality drift check.** Live-judge against production traffic; compare to baseline.
- **Cost drift check.** Per-request cost trend; alert on increases.

---

## 3. The lint stage

The lint stage runs cheap static checks. It is the first line of defense against trivial breakage.

### 3.1 What to lint

**Prompt structure validity.**

- Required sections present (system, examples, user-input slot).
- Required variables defined (`{user_input}`, `{retrieved_docs}`, etc.).
- No malformed templating ({{user_input)).
- Token-count sanity (a prompt > 10K tokens is a red flag).

**Schema validity.**

- Structured-output schemas parse as valid JSON Schema / Pydantic / Zod.
- Schema versions referenced in prompts match schemas in the prompt-store.
- No breaking schema changes without a version bump.

**Configuration validity.**

- Model versions in the release manifest are pinned, not aliased.
- Retrieval corpus version pinned.
- Eval suite version pinned.

**Secrets.**

- No API keys, no PHI, no PII in committed prompts. Pattern-based scanning.

**Forbidden constructs.**

- No model-alias references (`claude-opus-latest`).
- No environment-conditional logic in prompts (`if env == 'prod'`).
- No undocumented model-routing rules.

### 3.2 Lint tooling

- The prompt-store has a lint command. Run it in CI on every PR that touches `prompts/`.
- The release-manifest has a schema. Validate it on every PR.
- Pre-commit hooks (for engineers running locally) catch issues before push.

### 3.3 Lint as a fast-fail

Lint completes in < 30 seconds. If it fails, the PR fails immediately; the engineer fixes the issue before the fast eval (which takes much longer) is even invoked.

---

## 4. The fast eval

The fast eval is the per-PR quality check. It is a subset of the full eval, sized to run in under 10 minutes.

### 4.1 What goes in the fast eval

A *representative* subset of the golden set ([eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md)):

- 50–200 cases.
- Stratified across the feature's primary query types.
- Includes the 10–20 most-load-bearing cases (regression here = production incident).
- Includes safety / refusal canaries.

### 4.2 What doesn't go in the fast eval

- Stress / chaos cases (slow, rare; reserve for full eval).
- Long-tail rare-case coverage (slow, low-incremental-value at PR time).
- Cross-feature regression (slow; reserve for full eval).

### 4.3 The fast-eval gate

Per-PR criteria:

- Pass rate ≥ baseline - 1pp. (Allow small noise; block significant regression.)
- No critical-case failures. (The 10–20 load-bearing cases must all pass.)
- No safety regressions. (Safety canaries must all pass.)
- Cost on the eval suite within ±20% of baseline. (Cheap signal that prompt or model isn't blowing up cost.)

If any gate fails, the PR check fails. The engineer either fixes the issue or marks the PR as "intentional regression" with a documented justification (see §5.6).

### 4.4 Fast eval implementation

The fast eval runs as a CI job in the team's standard CI system. Same tooling, same dashboard, same notification flow as code tests.

- GitHub Actions / GitLab CI / Buildkite jobs.
- Output is a structured eval report attached to the PR.
- Pass/fail bubbled up to the PR check status.
- Slow-fail mode for unflaky jobs: rerun on intermittent failures, never on real signal.

### 4.5 Cost of the fast eval

A 100-case fast eval against an Opus-class model is ~10K tokens per case × $X/M tokens. At Opus pricing, $5–15 per fast-eval run. Multiplied across PRs, it is meaningful but bounded.

For high-PR-volume features, the fast eval can be conditioned: only run on PRs that touch `prompts/`, `eval/`, `models/`, or `corpus/`. PRs that touch only application code can skip it.

---

## 5. The full eval

The full eval is the deep check. Runs on every release candidate and on a schedule (nightly).

### 5.1 What goes in the full eval

The complete eval suite:

- All golden-set cases (1000s).
- Regression suites for every prior incident.
- Long-tail / rare-case coverage.
- Safety / refusal coverage.
- Cross-feature interaction tests (does changing the supervisor prompt break the drafting prompt's output format?).
- Cost / latency profiling.

### 5.2 The full-eval gate

Per-release criteria:

- Pass rate ≥ baseline - 0.3pp. (Tighter than fast-eval; this is the production gate.)
- No critical-case failures. Same as fast-eval.
- No safety regressions.
- No new regressions in the cross-feature suite.
- Cost per case within ±10% of baseline.
- Latency per case within ±15% of baseline.

If any gate fails, the release candidate fails. The team investigates before promoting.

### 5.3 Full-eval duration

Realistic durations:

- 100-case fast eval: < 10 minutes.
- 1000-case full eval: 1–4 hours.
- 10000-case full eval (large systems): overnight.

The full eval cost is meaningful — a 10K-case eval against a frontier model is hundreds of dollars per run. Budget it; run it on the schedule defined in [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md).

### 5.4 Eval result artifact

Every eval run produces a structured artifact:

```yaml
eval_run:
  id: 2026-05-25-r3-fulleval-2
  release_candidate: 2026.05.25-r3
  ran_at: 2026-05-25T03:14:22Z
  duration: 2h 14m
  cost: $214.32

  summary:
    cases_total: 1247
    cases_passed: 1239
    cases_failed: 8
    pass_rate: 99.36%
    baseline_pass_rate: 99.42%
    delta: -0.06pp

  critical_cases:
    total: 24
    passed: 24
    failed: 0

  safety_cases:
    total: 89
    passed: 89
    failed: 0

  failures:
    - case_id: med-term-normalization-047
      previous_status: pass
      new_status: fail
      output_diff: "..."

  cost_profile:
    mean_tokens_per_case: 4214
    mean_cost_per_case: $0.063
    baseline_mean_cost: $0.061
    delta: +3.3%

  latency_profile:
    p50_per_case_ms: 1840
    p95_per_case_ms: 4210
    baseline_p50: 1810
    baseline_p95: 4140
    delta_p50: +1.7%
    delta_p95: +1.7%

  artifacts:
    s3: s3://meridian-eval-archive/2026-05-25-r3/fulleval-2/
```

The artifact is checked into the release record. It is the audit trail.

### 5.5 Intentional regressions

Sometimes a regression is intentional — for example, a new safety policy that increases refusal rate by 2pp on edge cases that the team has decided should be refused. The eval gate must allow these.

The override mechanism:

- The PR or release author marks the regression as intentional in the eval-override file.
- They specify which cases / metrics are expected to regress and by how much.
- They provide a written justification.
- A senior reviewer (tech lead, product owner) approves the override.
- The override is logged in the release artifact.

Without the override, the gate fails. With the override, the gate passes — but the regression is *known and approved*, not hidden.

### 5.6 The flake floor

Eval suites have noise. Same model, same prompt, same case can produce different outputs (because temperature > 0, or because the judge has inherent variance). The gate threshold is set above the flake floor:

- Measure the flake floor by running the same eval twice on the same artifact. Difference is the floor.
- Set the gate threshold above 2× the flake floor.
- Re-measure quarterly; the floor moves when judge or rubric changes.

A gate set below the flake floor fails on noise, gets bypassed, and becomes meaningless.

---

## 6. The cost and latency gates

Quality is not the only dimension. Cost and latency get their own gates.

### 6.1 The cost-regression gate

Computed during the full eval:

- Mean cost per case in the candidate.
- Mean cost per case in the baseline.
- Delta.

Gate criterion: delta ≤ +10% (the threshold is tunable per workload).

Why a separate gate: a prompt change that improves quality 0.2pp but increases cost 40% is rarely worth shipping. The cost gate catches this before promotion.

### 6.2 The latency-regression gate

Computed during the full eval:

- p50 / p95 latency per case in candidate.
- p50 / p95 latency in baseline.
- Delta.

Gate criterion: p95 delta ≤ +15% and p50 delta ≤ +10% (tunable).

Why a separate gate: a model swap to a slower model is sometimes the right call, but it should be an explicit decision, not an accident. The latency gate makes the decision explicit.

### 6.3 Override mechanism for cost and latency

Same as the eval override: intentional regressions are allowed with documented justification and senior approval. Without the override, the gate fails.

### 6.4 What's not in the gate (yet)

- Token-count regression (a prompt that doubles in length without quality improvement). Worth tracking but harder to gate cleanly; treat as a code-review concern.
- Per-tenant cost shifts. Detected post-promote in production cost dashboards.

---

## 7. The canary stage

Even with eval gates green, the production traffic distribution may differ enough from the eval distribution that something the eval did not catch shows up in production. The canary catches it before full-traffic exposure.

### 7.1 The canary mechanic

See [model-lifecycle/canary-and-shadow-rollout.md](../model-lifecycle/canary-and-shadow-rollout.md) for the per-model canary mechanics. The CI/CD layer:

- Triggers the canary deploy after eval / cost / latency gates pass.
- Starts at 1% of traffic to the new version.
- Monitors against canary criteria for a defined window (1–24 hours, workload-dependent).
- Either ramps to the next level (10%, 50%, 100%) or rolls back.

### 7.2 Canary criteria

The criteria are the same shape as the eval gate, but read from production telemetry rather than offline eval:

- Live-judge quality on canary traffic ≥ baseline - 0.5pp (more permissive than offline eval because the sample is smaller).
- Error rate on canary ≤ baseline + 0.2pp.
- p95 latency on canary ≤ baseline + 20%.
- Cost per request on canary ≤ baseline + 15%.
- No safety incidents traceable to canary.

### 7.3 Canary duration per ramp step

Workload-dependent. Faster for high-volume features (more data per hour); slower for low-volume features.

- 1% step: ≥ 1000 sessions or 1 hour, whichever is longer.
- 10% step: ≥ 10K sessions or 4 hours.
- 50% step: ≥ 50K sessions or 12 hours.
- 100% step: full promotion; existing version retained for 14 days for rollback.

### 7.4 Automated ramp vs human-gated ramp

- Low-risk changes (sub-version refresh of a known model): automated ramp on criteria.
- High-risk changes (model swap, fine-tune deployment, major prompt rewrite): human gate at each ramp step.

The pipeline supports both. The default for production-impacting AI changes is human-gated for the first canary step, automated for the rest.

---

## 8. The promote stage

When 100% traffic is on the new version for the holding window without incident, the version is promoted:

- The previous version's pin is moved to "rollback target."
- The release artifact is finalized and archived.
- The deploy event is logged with full version metadata.
- Eval gate baselines update to the new version's metrics (so the next PR's "is this a regression" check uses the new baseline).
- The old version's eval and canary data are archived (the release artifact references them for posterity).

The old version's runtime resources (GPU instances, prompt store entries, etc.) remain available for the rollback window (typically 14 days), then are reclaimed.

---

## 9. The branch protection model

Branch protection is what enforces the pipeline. Without it, the pipeline is a guideline.

### 9.1 Protected branches

- `main` is protected.
- `release/*` branches are protected.

### 9.2 Required checks for PRs

To merge to `main`:

- Lint passes.
- Fast eval passes.
- At least one reviewer approves.
- If the PR touches `prompts/`, `models/`, `corpus/`, or `eval/`: a second reviewer from the AI Platform team approves.

### 9.3 Required gates for deploy

To deploy a release:

- Release candidate passes full eval.
- Cost-regression gate passes.
- Latency-regression gate passes.
- The release artifact is signed by the release author *and* an approver.

### 9.4 Override discipline

Overriding any gate requires:

- Documented justification in the PR or release.
- Approval from a specific role (tech lead for fast-eval override; tech lead + product owner for full-eval override; SRE on-call for cost/latency override).
- Override logged in the release artifact.

Overrides are not banned — sometimes the right call is to ship despite a gate failure. But every override is recorded; the team can audit overrides quarterly to see if any pattern of overrides suggests a gate is mis-tuned.

### 9.5 Emergency hotfix path

For genuine production emergencies (safety incident, severe outage), an emergency hotfix can bypass the standard pipeline:

- Emergency hotfixes still go through lint.
- Fast eval is run but failure does not block the hotfix.
- Full eval is run in the background; if it fails, the hotfix is rolled back.
- Canary is 1% for 15 minutes (not 1 hour); ramp is faster.
- Emergency hotfix authority is the SRE on-call lead plus the product owner.
- Every emergency hotfix is reviewed in the post-incident retro.

The emergency path exists to be used sparingly. If it is used more than once a quarter, the standard pipeline is too slow and needs investment.

---

## 10. Worked Meridian example: shipping a Care Coordinator prompt update

The Care Coordinator's drafting prompt has accumulated drift: clinicians have requested several wording tweaks over six weeks. The AI Platform engineer consolidates the changes into a single prompt update and ships it through the pipeline.

### 10.1 PR

- Engineer opens PR with the prompt change.
- Lint runs in 22 seconds: prompt structure valid, no aliases, no secrets, schema unchanged. Pass.
- Fast eval runs in 8 minutes: 150 stratified golden-set cases, runs against the candidate prompt + current pinned model.
  - 148/150 pass. Baseline pass rate: 99.1%. Candidate: 98.7%. Delta: -0.4pp.
  - Critical cases: 18/18 pass.
  - Safety canaries: 12/12 pass.
  - Cost per case: $0.061 baseline, $0.063 candidate (+3.3%). Within tolerance.
  - Fast-eval gate: PASS.
- Two reviewers approve (one clinical content reviewer, one AI Platform reviewer).
- PR merges.

### 10.2 Release candidate

- Post-merge, CI builds release candidate `2026.05.25-r3`.
- Full eval runs nightly: 1247 cases.
  - 1239/1247 pass. Baseline: 99.42%. Candidate: 99.36%. Delta: -0.06pp.
  - Critical cases: 24/24 pass.
  - Safety: 89/89 pass.
  - Cross-feature: pass.
  - Cost: $0.061 → $0.063 (+3.3%). Pass.
  - Latency: p95 4140ms → 4210ms (+1.7%). Pass.
- Eval artifact archived.
- Full-eval gate: PASS.

### 10.3 Canary

- Canary deploys at 1% Care Coordinator traffic to `2026.05.25-r3`.
- Monitoring window: 4 hours.
  - Live-judge quality: 7.42 baseline → 7.45 canary. No regression.
  - p95 latency: 1340ms baseline → 1320ms canary. No regression.
  - Cost per conversation: $0.082 → $0.084. +2.4%. Within tolerance.
  - Refusal rate: 1.1% → 1.0%. No regression.
  - Error rate: 0.04% → 0.03%. No regression.
- Canary 1% gate: PASS. Auto-ramp to 10%.
- 10% canary: monitored 4 hours. PASS. Auto-ramp to 50%.
- 50% canary: monitored 12 hours. PASS. Human-gated promote step requires AI Platform lead approval.
- AI Platform lead reviews dashboard, approves.
- Promote to 100%.

### 10.4 Promotion

- 100% traffic on `2026.05.25-r3`.
- 14-day rollback retention armed.
- Eval baselines updated.
- Release artifact archived.

### 10.5 Time from PR-open to 100%

- PR open → merge: 6 hours (two reviewers needed clinical-content review).
- Merge → full eval complete: 4 hours.
- Full eval → canary 100%: 22 hours (1% → 10% → 50% → human-gate → 100%).
- Total: 32 hours.

A change that previously shipped in minutes (notebook-edit-and-deploy) now takes 32 hours. The trade: 32-hour latency for a process that catches a class of regressions the previous flow missed.

### 10.6 Findings closed

- **ARCH-CARE-060** (prompts shipped without PR; no review record).
- **ARCH-CARE-061** (no eval gate on prompt changes; regressions slipped through).
- **ARCH-CARE-062** (no canary on prompt changes; full-traffic exposure on first deploy).
- **ARCH-CARE-063** (no rollback path; "rollback" meant editing the prompt back in the notebook).
- **ARCH-CARE-064** (no audit trail; "which prompt was running last Tuesday?" was unanswerable).

---

## 11. Anti-patterns

### 11.1 The notebook-edit-and-ship

The engineer edits a prompt in a notebook, copies it into the prompt store via an admin UI, and traffic hits the new prompt within minutes. No PR, no review, no eval, no canary.

The fix: branch protection on the prompt store. The store accepts changes only from approved release pipelines, not from manual edits.

### 11.2 The eval gate that nobody can pass

The gate threshold is set too tight (e.g., zero pass-rate regression allowed) and trips on flake. The team learns to bypass the gate. Within a quarter the gate is decoration.

The fix: set the threshold above 2× the flake floor. Measure the floor. Re-tune quarterly.

### 11.3 The eval gate that nobody fails

The gate threshold is set too loose (e.g., pass-rate ≥ 50%). Every change passes; the gate provides no signal. The team trusts the gate; production incidents continue.

The fix: set the threshold near the historical pass-rate floor for the suite. A passing gate should mean "no regression," not "the system still half-works."

### 11.4 The bypassed canary

The team ships hotfixes by deploying directly to 100% "because it's urgent." Within months, "urgent" is the default and canary is theater.

The fix: emergency-hotfix path is real but auditable. Quarterly review of emergency hotfixes. If the count rises, the standard pipeline is too slow and needs investment.

### 11.5 The "we'll add the gate later" promise

The team ships the pipeline without the eval gate, intending to add it later. Six months later, the gate is still not added; the pipeline is shape-of-pipeline without substance.

The fix: the eval gate is the load-bearing step. Ship the pipeline with it or do not call it a pipeline.

### 11.6 The cost gate that fires on noise

Cost-per-case has natural variance. The gate is set at +10% and fires on PRs that did nothing to cost (just hit a noisy run). Engineers learn to override; the gate becomes meaningless.

The fix: aggregate the cost signal across enough cases that the per-PR variance is small. If 100 cases is too few, run more.

### 11.7 The cross-tenant unbenchmark

Some PRs change behavior for one tenant only (per-tenant prompt). The fast eval covers the default behavior; the per-tenant behavior is unbenched. Regressions land for that tenant unnoticed.

The fix: per-tenant eval suites for the tenants where behavior diverges. The fast eval runs the default; the full eval runs the per-tenant set.

### 11.8 The release artifact that is missing the eval result

The release manifest ships without the eval-pass-result. Three weeks later, an incident review asks "what did the eval say at release?" and nobody can answer.

The fix: the eval result is part of the release artifact. The artifact is incomplete without it. The release pipeline refuses to promote an artifact missing the eval result.

---

## 12. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| CICD-PIPE-001 | AI changes (prompts / models / corpora) deploy outside CI/CD | High | Branch-protect AI artifacts; require PR + pipeline | AI Platform + SRE |
| CICD-PIPE-002 | No lint stage for prompts / configs | Medium | Add lint job to CI; gate per §3.1 | AI Platform |
| CICD-PIPE-003 | No fast eval on PR; full eval only at release | High | Add fast eval (50–200 cases, < 10 min) per PR; bubble up to PR check status | AI Platform + Eval Eng |
| CICD-PIPE-004 | Eval gate threshold not anchored to flake floor | High | Measure flake floor; set threshold above 2× floor; re-measure quarterly | Eval Eng |
| CICD-PIPE-005 | No cost-regression gate; cost-increasing changes ship invisibly | High | Add cost gate per §6.1; integrate with [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) | AI Platform + FinOps |
| CICD-PIPE-006 | No latency-regression gate | Medium | Add latency gate per §6.2 | AI Platform + SRE |
| CICD-PIPE-007 | Canary skipped or set at >10% for new versions | High | Default 1% → 10% → 50% → 100% per §7.3 | AI Platform + SRE |
| CICD-PIPE-008 | Override mechanism undocumented; gate bypasses untracked | High | Document override path per §5.5 and §9.4; log every override in release artifact | AI Platform |
| CICD-PIPE-009 | Eval results stored in spreadsheet / chat, not in repo | Medium | Eval result is part of release artifact ([release-artifacts-for-ai.md](./)) | Eval Eng + AI Platform |
| CICD-PIPE-010 | No branch protection on prompt store / model registry | High | Branch-protect; merges only via pipeline | AI Platform + SRE |
| CICD-PIPE-011 | Emergency hotfix path absent or unaudited | Medium | Define emergency path per §9.5; quarterly audit of use | SRE + AI Platform |
| CICD-PIPE-012 | Fast eval cost not tracked; high-PR-volume team burns budget | Low | Track fast-eval cost; condition on PR scope (skip on app-only changes) | FinOps + AI Platform |
| CICD-PIPE-013 | Per-tenant behavior not covered by fast eval | Medium | Per-tenant eval subsets for tenants with divergent behavior | AI Platform + Eval Eng |
| CICD-PIPE-014 | Full eval cost not budgeted; nightly runs blow finance estimates | Low | Budget full-eval cost; right-size suite via [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) | FinOps + Eval Eng |
| CICD-PIPE-015 | Canary criteria identical to eval criteria; production-mix regressions slip | Medium | Define separate canary criteria from production telemetry per §7.2 | AI Platform + SRE |
| CICD-PIPE-016 | Rollback target not preserved across promotion | High | 14-day retention of previous version + pin; verify weekly | AI Platform + SRE |
| CICD-PIPE-017 | PR checks not bubbled to PR UI; reviewers don't see eval status | Medium | Wire fast-eval result to PR check status; require green to merge | AI Platform |
| CICD-PIPE-018 | Pipeline shape exists but eval gate is decorative (gate threshold too loose) | High | Re-tune gate per §5.6 and §11.3; ensure passing gate means "no regression" | Eval Eng + AI Platform |

---

## 13. Adoption checklist

- [ ] Branch protection on `main` and release branches; protection on prompt store and model registry.
- [ ] Lint stage runs per PR (< 30s); prompt validity, schema validity, no aliases, no secrets, no forbidden constructs.
- [ ] Fast eval runs per PR (< 10 min); 50–200 cases including critical and safety canaries.
- [ ] Fast-eval gate threshold above flake floor; re-tuned quarterly.
- [ ] Full eval runs on every release candidate and nightly on `main`.
- [ ] Eval results captured in structured release artifact.
- [ ] Cost-regression gate active per release.
- [ ] Latency-regression gate active per release.
- [ ] Canary defaults to 1% → 10% → 50% → 100% with monitoring windows.
- [ ] Canary criteria distinct from eval criteria (production telemetry, not offline metrics).
- [ ] Human gate at first canary step for high-risk changes; automated for low-risk.
- [ ] Override mechanism with documented justification and senior approval; overrides logged.
- [ ] Emergency hotfix path defined and audited quarterly.
- [ ] Rollback target retained for 14 days; verified weekly.
- [ ] All gate failures and overrides bubbled to PR / release UI.
- [ ] PR template prompts the author to identify which artifacts changed (prompts / models / corpora / code).

---

## 14. References

**Internal:**

- [eval-gate-design.md](./eval-gate-design.md) — the load-bearing gate at the heart of this pipeline.
- [prompt-version-pinning.md](./prompt-version-pinning.md) — the pinning discipline this pipeline enforces.
- [model-version-pinning.md](./model-version-pinning.md) — the pinning discipline for models.
- [release-artifacts-for-ai.md](./) — the artifact format the pipeline produces (coming).
- [canary-rollouts.md](./) — canary mechanics for AI changes (coming, this pipeline's deployment stage).
- [shadow-traffic.md](./) — the shadow alternative to canary (coming).
- [model-lifecycle/model-promotion.md](../model-lifecycle/model-promotion.md) — the model-side deployment workflow.
- [model-lifecycle/canary-and-shadow-rollout.md](../model-lifecycle/canary-and-shadow-rollout.md) — model-level canary mechanics.
- [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md) — the rollback path the pipeline preserves.
- [model-lifecycle/ab-model-testing.md](../model-lifecycle/ab-model-testing.md) — A/B as a comparison technique downstream of the pipeline.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — upstream eval design.
- [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — the suite definitions the pipeline runs.
- [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — the canary's quality signal.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — the cost signal for the gate.
- [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) — production cost monitoring for post-promotion.
- [prompt-engineering/prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md) — the prompt-side discipline this pipeline enforces.
- [prompt-engineering/prompt-versioning.md](../prompt-engineering/prompt-versioning.md) — versioning the prompts the pipeline pins.

**Cross-repo (architecture sibling):**

- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — worked-example system.
- [reference-systems/adoption-sequencing-across-systems.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/adoption-sequencing-across-systems.md) — adoption sequence including CI/CD discipline.
