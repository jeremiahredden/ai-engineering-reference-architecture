# Model Lifecycle

## What this folder is

The engineering practice of managing models as first-class production dependencies — registry, promotion, fine-tuning operations, deployment, A/B and canary rollout, rollback, and the operational pattern for migrating off models that are sunset or deprecated. The material here is what I put in front of a team when the question is: *we have eight different model names referenced across the codebase, one of them is being deprecated in 90 days, two more were quietly replaced when the SDK auto-upgraded — how do we get model versions under control?*

## The organizing principle

Models are dependencies. In 2026 that means three things simultaneously: (1) the model is a *versioned external service* with deprecation dates, breaking-change risk, and a vendor's release schedule that the team does not control; (2) the model is a *unit of behavior* — switching a model is closer to switching a library that defines core business logic than to switching a back-end implementation that should be transparent; and (3) the model is the largest single cost line in the system, so model choice has financial as well as behavioral consequences.

Most teams treat models the way teams in 2015 treated container base images — as something pulled by name at runtime, with no pinning, no inventory, no change control, and a cycle of surprises every quarter when something silently changes underneath. The patterns here treat models as catalogued dependencies with the same operational discipline already applied to libraries and to base images.

The folder is opinionated about three things specifically. First, that every model in production should be *pinned* to a specific version, not referenced by alias — the `claude-opus-latest`-style aliases are convenient and dangerous. Second, that model migrations are *projects*, not inline swaps — every model change goes through the same eval / canary / rollback pipeline that code changes go through. Third, that fine-tuning is the most expensive lifecycle pattern to maintain and should be the last lever pulled, not the first.

## Planned documents

- **model-registry.md** *(coming)* — The platform pattern of a central model registry: every model in use, its version pin, owner team, allowed contexts, BAA / regulatory coverage status, deprecation date, and per-model usage telemetry. Build-vs-buy (MLflow, internal registry, vendor catalogues), the schema, and the integration with deployment so out-of-registry models cannot ship.
- **[model-promotion.md](./model-promotion.md)** — The dev → staging → prod promotion path for model versions, the eval gate that blocks promotion on regression, the per-environment configuration discipline, and the model-version-as-a-release-artifact pattern.
- **[fine-tuning-operations.md](./fine-tuning-operations.md)** — When fine-tuning earns its operational cost, the data pipeline for fine-tune datasets, the fine-tune-as-CI-job pattern, the eval comparison against the base model, version control of fine-tuned models, and the deprecation playbook when the base model is replaced.
- **[distillation-operations.md](./distillation-operations.md)** — Distilling a larger frontier model into a smaller open-weight model for cost / latency. When the trade-off is worth it, the data-collection pattern (frontier-as-teacher on production traffic samples), the eval pattern that validates the distillate, and the redistill-on-model-update lifecycle.
- **[canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md)** — Canary rollout (small percentage of real traffic to new model, monitor quality / cost / latency, ramp), shadow traffic (full traffic to new model in parallel, compare outputs, no user impact), the cost-of-shadow trade-off, and the rollback criteria.
- **[ab-model-testing.md](./ab-model-testing.md)** — Running an A/B between two models in production, the statistical-significance design for noisy quality metrics, integration with online evals, and the interaction with cost and latency that makes "winner" a multi-dimensional rather than single-dimensional question.
- **[rollback-procedures.md](./rollback-procedures.md)** — Model-rollback runbook: detection (quality alert, cost alert, user complaints), decision (rollback vs hot-patch), execution (pinned-version rollback through the registry), and the post-incident review. The "no rollback path" anti-pattern that turns a quality issue into an extended outage.
- **[model-deprecation-playbook.md](./model-deprecation-playbook.md)** — The structured project for migrating off a model that has been sunset by its vendor: timeline (typical 6 months from announcement to shutdown), parallel-running with the candidate replacement, eval cross-check, prompt-port discipline, the rollout sequence, and the rollback criteria. Includes the worked example of migrating Meridian Care Coordinator from a previous Claude generation to the current one.
- **[runtime-platform.md](./runtime-platform.md)** — For self-hosted open-weight models: vLLM, TGI, SGLang, Triton, the throughput / latency / memory trade-offs, the per-model deployment shape (dedicated instance vs shared multiplexed serving), autoscaling for spiky workloads, and the operational cost of running inference infrastructure yourself.

## How to use this section

**If your codebase references models by alias** (`gpt-4-latest`, `claude-3-opus`), `model-registry.md` and `model-promotion.md` together are the refactor. The fix is "pin versions, route through registry"; the operational pattern is the platform discipline that keeps it pinned.

**If you have a model deprecation deadline approaching**, `model-deprecation-playbook.md` is the runbook. Treat it as a structured project, not an inline swap.

**If you are considering fine-tuning**, `fine-tuning-operations.md` is the honest accounting of what the practice will cost long-term. Most teams underestimate the maintenance load and over-estimate the capability gap.

## What this section is not

- **An ML training reference.** General ML training (loss curves, optimizer choice, mixed-precision training, distributed training, hyperparameter search) is the ML engineering canon. This folder is about the *operations* around the trained / fine-tuned / chosen models, not about training methodology.
- **A model-selection guide.** Which model to pick for which workload is the architecture sibling's `model-strategy/` folder. This folder is about operating whichever models you have selected.
