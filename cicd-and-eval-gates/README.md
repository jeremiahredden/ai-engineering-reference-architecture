# CI/CD and Eval Gates

## What this folder is

The engineering practice of treating AI features as code that ships through a CI/CD pipeline with the same rigor as the rest of the codebase — with the addition that the pipeline includes eval gates, prompt-version pinning, model-version pinning, and AI-specific canary patterns. The material here is what I put in front of a platform team when the question is: *the rest of the company ships changes through a pipeline with tests, linting, and approval gates; AI changes ship by editing a prompt in a notebook — how do we close the gap?*

## The organizing principle

AI features have outsized blast radius per change. A one-word change to a system prompt can shift quality across every interaction the feature serves. A model-version bump can change tone, format, refusal behavior, and tool-call selection in ways that no unit test will catch. A retrieval-corpus refresh can silently degrade the answers on a subset of queries. These changes need the same change-control discipline as a database migration or an API breaking change — and they almost never get it.

So the patterns here apply the discipline the team already practices on code (PRs, reviews, CI gates, canary rollout, rollback) to the AI-specific artifacts (prompts, models, datasets, evals). The mechanics are the same; the artifacts and the gate criteria are different. An eval gate that fails CI on a quality regression is structurally the same as a unit-test gate that fails CI on a logic regression — the only differences are that the eval gate is slower, more expensive, and noisier.

The folder is opinionated about three things specifically. First, that *every AI change goes through CI* — prompt changes, model-version changes, retrieval-corpus updates, fine-tune deployments. The notebook-edit-and-ship pattern is the source of most production AI incidents. Second, that *the eval gate is the load-bearing CI step for AI changes* — without it the rest of the pipeline is decoration. Third, that *AI rollouts default to canary or shadow*, not direct cutover, because the failure modes are hard to detect in pre-production.

## Planned documents

- **[pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md)** — The CI/CD pipeline shape for AI changes: lint (prompt structure, schema validity), fast eval (subset that runs per PR in < 10 minutes), full eval (runs nightly and on release candidates), cost-regression check, latency-regression check, canary deploy, monitor-and-promote. The branch-protection rules that prevent merge without the gates passing.
- **[eval-gate-design.md](./eval-gate-design.md)** — Where the eval gate sits, the threshold-setting discipline (block on pass-rate drop > X%), the fast-subset selection for per-PR runs, the full-suite scheduling, the integration with the sibling `eval-engineering/` folder, and the override-with-justification pattern for intentional regressions.
- **[prompt-version-pinning.md](./prompt-version-pinning.md)** — Pinning the prompt version in the release artifact (alongside code version and model version), the prompt-store-as-dependency pattern, the deployment integration that fails if the pinned prompt version is missing, and the rollback that includes prompt rollback alongside code rollback.
- **[model-version-pinning.md](./model-version-pinning.md)** — Same discipline for model versions: pin the specific model version in the release, fail deployment if the pinned model is unavailable, route through the model registry (sibling `model-lifecycle/` folder) for promotion. The deny-list pattern that prevents accidental use of unapproved or deprecated models.
- **[dataset-version-pinning.md](./dataset-version-pinning.md)** — Pinning dataset versions in releases for systems where the dataset is part of the system behavior (retrieval corpora, fine-tune data, eval golden sets). The reproducibility benefit: a given commit can be exactly reproduced because code, prompt, model, and data versions are all pinned.
- **[canary-rollouts.md](./canary-rollouts.md)** — Canary deployment for AI changes: small-percentage traffic to the new version, automated quality / cost / latency monitoring on the canary, automated promote or rollback based on thresholds, and the integration with online evals. The 1% → 10% → 50% → 100% ramp pattern adapted for AI workloads.
- **[shadow-traffic.md](./shadow-traffic.md)** — Running new prompts / models in shadow against production traffic without user impact: full-traffic to both, capture outputs from both, compare offline, no rollback risk. The cost trade-off (shadow doubles the AI bill on the shadowed traffic), when it is worth it, and when canary is sufficient.
- **[release-artifacts-for-ai.md](./release-artifacts-for-ai.md)** — The release artifact contents for an AI service: code version, prompt versions, model versions, dataset versions, eval-suite version, eval-pass-results, cost-baseline. The reproducibility guarantee, the rollback target, and the audit-trail role.

## How to use this section

**If your AI changes do not go through CI**, `pipeline-architecture-for-ai.md` is the design and `eval-gate-design.md` is the load-bearing implementation. The minimum-viable pipeline is "PR → fast eval → review → merge → canary → monitor → promote."

**If your AI rollouts are direct-cutover**, `canary-rollouts.md` is the upgrade. The canary investment pays back the first time it catches a regression that would otherwise have shipped at 100% traffic.

**If your releases are not reproducible** ("we are not sure which prompt version was running last week when the incident happened"), the version-pinning trio (`prompt-version-pinning.md`, `model-version-pinning.md`, `dataset-version-pinning.md`) is the fix.

## What this section is not

- **A CI/CD primer.** General CI/CD practices (GitHub Actions / GitLab / Jenkins, branching strategy, environment promotion, deployment automation) are well-covered elsewhere. This folder is about the AI-specific gates and artifacts.
- **A vendor tooling guide.** Where CI tools and AI-eval-platforms are named, they are illustrative; the practice is what matters.
