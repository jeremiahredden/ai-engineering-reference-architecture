# Runtime Platform (Self-Hosted Inference)

> **Audience.** Engineers building or operating self-hosted inference for open-weight or fine-tuned models. Platform teams whose first deployment ran on a single GPU box and now has user-facing latency problems. SREs whose runbook for AI inference reads "talk to a vendor" and would like a real one. **Scope.** The *engineering* practice of running a model runtime in your own infrastructure: serving-stack selection (vLLM, TGI, SGLang, Triton); throughput / latency / memory trade-offs; per-model deployment shape (dedicated vs shared multiplexed); autoscaling for spiky AI workloads; the operational cost of running inference infrastructure yourself; integration with the model registry and CI/CD. Pair with [model-registry.md](./model-registry.md) (the catalog) and [model-promotion.md](./model-promotion.md) (the deployment flow). Cross-link to [reliability-engineering/capacity-planning.md](../reliability-engineering/capacity-planning.md) (the capacity model) and to [reliability-engineering/multi-provider-failover.md](../reliability-engineering/multi-provider-failover.md) (the fallback when self-hosting falters). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Most teams that talk to a model talk to a vendor's hosted API. The vendor handles autoscaling, hardware, runtime selection, batching, GPU choice, and the operational burden. The team writes a JSON body and reads a JSON response.

Some teams need to self-host: data residency constraints that rule out vendor APIs, cost at scale that crosses over the build-vs-buy line, fine-tuned models on weights the vendor will not host, latency profiles that the vendor cannot meet, or open-weight model strategies where the model is the team's IP. In any of those cases, the team becomes responsible for an *inference platform* — and the gap between "I have weights on disk" and "I have a production inference service" is much larger than it looks.

The patterns in this document are the same patterns a serving infrastructure team would learn over 6–12 months of operating a runtime in production, condensed. The serving stack (vLLM, TGI, SGLang, Triton) is one decision. The per-model deployment shape (dedicated vs shared) is another. The autoscaling pattern for spiky AI workloads is a third — and is meaningfully different from autoscaling web services. The operational cost (engineers, GPU spend, observability investment) is the fourth and is the one most teams underestimate.

The honest framing: self-hosting an LLM is not "we'll just deploy the weights." It is a platform investment. Done well, it gives the team capabilities the vendor APIs cannot match. Done poorly, it produces an under-performing service that costs more than the vendor would have.

This document is opinionated about four things:

1. **Self-host only when the requirement is real.** Data residency, cost-at-scale crossover, fine-tune on un-hostable weights — these are real reasons. "We want to avoid vendor lock-in" is not, by itself, a real reason; the build-vs-buy decision ([model-strategy/build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md)) is the prerequisite.
2. **Pick the serving stack on workload shape, not benchmark numbers.** A stack that's 20% faster on a benchmark may be the wrong fit for a multi-tenant workload that benefits from continuous batching but suffers from head-of-line blocking.
3. **Default to shared multiplexed serving; reach for dedicated only when isolation is required.** Dedicated-per-model is expensive; shared-across-models requires more sophisticated routing but utilizes hardware better.
4. **Plan for the operational cost. Two FTE on the inference platform is the floor, not the ceiling.** A team that "just runs vLLM behind nginx" does not have a production platform; they have a fragile prototype.

Structure: (2) when self-hosting is the right call; (3) the serving stacks; (4) throughput vs latency vs memory; (5) deployment shape (dedicated vs shared); (6) autoscaling for AI workloads; (7) the GPU choice; (8) operational cost; (9) integration with model registry and CI/CD; (10) worked Meridian example; (11) anti-patterns; (12) findings; (13) adoption checklist; (14) references.

---

## 2. When self-hosting is the right call

Self-hosting is the right call when at least one of these holds:

### 2.1 Data residency

Workloads that cannot leave a specific jurisdiction or a customer's VPC. Healthcare data with PHI under HIPAA when the vendor's BAA does not cover the workload, financial data with national-residency rules, defense workloads that require on-premise inference. The data constraint forces the runtime to come to the data, not the reverse.

For Meridian, certain customers in EU and APAC regions have data-residency contracts that rule out US-hosted vendor APIs. The clinical-knowledge model for these tenants must run in-region. Self-hosting in those regions becomes the only path.

### 2.2 Cost-at-scale crossover

The vendor's per-token pricing becomes uneconomic past some volume. The crossover depends on the model size, the GPU, and the team's utilization. A 70B open-weight model on rented H100s can be cheaper than a comparable frontier model on a vendor API somewhere between 1B and 10B tokens per month, depending on cost negotiations and steady-state utilization.

The crossover is workload-specific. The build-vs-buy doc covers the analysis; this doc is what happens *after* the analysis says to build.

### 2.3 Fine-tuned weights on un-hostable models

The team has fine-tuned an open-weight base. The vendor that hosts the base cannot host the fine-tune (no fine-tune-hosting product or no support for the specific base). The team self-hosts the fine-tune.

### 2.4 Latency profiles vendor APIs cannot meet

Some workloads need sub-50ms first-token latency to a model. Vendor APIs typically deliver 200–800ms first-token over the public internet. Self-hosting in a peered VPC or on-prem can deliver the lower latency at the cost of operating the platform.

### 2.5 IP control

The model is the team's competitive advantage and they do not want the weights on a vendor's hardware. Common in research-derived models, proprietary fine-tunes, and regulated industries where the model and its training data are treated as protected IP.

### 2.6 When self-hosting is the *wrong* call

- **"We want to avoid vendor lock-in"** as the only reason. The cost of building the platform is greater than the lock-in risk for most teams.
- **"We can save money."** Without a serious volume baseline, the operational cost (FTEs + GPU rental + observability) typically exceeds the vendor bill at the scale of teams asking the question.
- **"We need lower latency on a single call."** A single low-latency call is achievable on vendor APIs with regional endpoints. Self-host only if the *aggregate* latency profile of a high-throughput workload demands it.

---

## 3. The serving stacks

Four open-source serving stacks dominate in 2026: vLLM, TGI, SGLang, and Triton. Each has a clear sweet spot.

### 3.1 vLLM

The most popular, most-developed open-source LLM serving stack. Designed around PagedAttention, which makes continuous batching of variable-length sequences efficient.

Strengths:

- Throughput-optimized for batched workloads.
- Strong support for the widest range of open-weight models (Llama, Mistral, Qwen, DeepSeek, Granite, Phi, Gemma).
- Mature continuous batching; handles mixed prompt lengths gracefully.
- Active community; new models supported within days of release.

Weaknesses:

- Latency on a single request is *not* its design goal. p50 on an unloaded vLLM instance is higher than on simpler runtimes.
- Memory-management overhead has historically required tuning.
- Production hardening (graceful shutdown, structured logging, prometheus metrics) has improved but is still less mature than long-running production services.

When to use vLLM: most general-purpose batched serving. The default choice for a new self-hosted deployment unless something specific argues against it.

### 3.2 TGI (Text Generation Inference, Hugging Face)

A serving stack originally built for the Hugging Face inference endpoints and now widely used in production.

Strengths:

- Production-ready out of the box (metrics, health endpoints, graceful shutdown).
- Tight Hugging Face ecosystem integration (model loading, tokenizers, weights formats).
- Good support for safetensors and weight quantization (AWQ, GPTQ, bitsandbytes).
- Streaming responses are first-class.

Weaknesses:

- Throughput is generally lower than vLLM on continuous-batching workloads.
- Model support is slightly behind vLLM for the bleeding edge.
- The license has historically had restrictions; verify before commercial use.

When to use TGI: when production hardening and Hugging Face ecosystem fit are more important than peak throughput; when the team's deployment pattern is already Hugging Face oriented.

### 3.3 SGLang

A newer stack focused on structured generation and multi-step reasoning patterns. The runtime is co-designed with a programming language (also called SGLang) for expressing complex inference patterns.

Strengths:

- Optimized for workloads with structured output, constrained decoding, and tool-use loops.
- RadixAttention prefix-cache reuse can yield substantial speedups when prompts share large prefixes (system prompts, few-shot examples, RAG context).
- Multi-step inference (chain-of-thought, tool-use loops) is first-class.

Weaknesses:

- Newer; fewer production deployments than vLLM or TGI.
- Smaller community; less battle-tested operational tooling.
- Best fit is a narrower workload (heavy structured generation, agentic loops, shared-prefix workloads).

When to use SGLang: agentic workloads with shared prefixes; RAG with large system prompts; workloads with substantial structured-output volume. Especially good for Care-Coordinator-shaped systems with a large system prompt and many short conversations.

### 3.4 Triton (NVIDIA Triton Inference Server)

A general-purpose inference server, not LLM-specific, supporting many model types (LLMs, vision models, traditional ML).

Strengths:

- Production maturity (years in production at NVIDIA-customer scale).
- Multi-model serving across model types from one server (LLM + embedding + reranker + vision in one runtime).
- Strong NVIDIA hardware integration; TensorRT-LLM backend for optimized inference.
- Detailed metrics and observability.

Weaknesses:

- More complex to configure than vLLM or TGI.
- LLM-specific features (continuous batching, paged attention) historically lagged behind LLM-specific runtimes; closing fast.
- Some advanced features require TensorRT compilation which adds a build step.

When to use Triton: when the deployment serves multiple model *types* (not just LLMs), when the team is already on NVIDIA-optimized stacks (TensorRT, CUDA), or when production maturity at scale is the load-bearing requirement.

### 3.5 The decision

| Workload | Default stack |
|---|---|
| General batched LLM serving | vLLM |
| Production-first, Hugging Face ecosystem | TGI |
| Heavy structured generation, agentic, shared prefixes | SGLang |
| Multi-model-type platform, NVIDIA-deep | Triton |
| Mixed (LLM + embedding + reranker), high scale | Triton (or two stacks: vLLM for LLMs + a separate embedding runtime) |

For most teams self-hosting for the first time: start with vLLM. It is the safest default, the largest community, and the largest fund of operational knowledge to draw from when something breaks.

---

## 4. Throughput vs latency vs memory

The three dimensions of inference performance trade against each other. Understand the trade.

### 4.1 Throughput

Tokens per second across all concurrent requests. Maximized by:

- **Continuous batching.** Group multiple requests into one batch, dispatch through the model once, send responses back per-request.
- **Large batch sizes.** Bigger batches saturate the GPU better but increase per-request latency.
- **Long sequences.** Long-sequence kernels are more compute-bound and use the GPU more efficiently than short-sequence kernels.

Workloads optimized for throughput: bulk processing (overnight summarization, batch transcript analysis), large-scale embedding generation, anything in `cost-and-finops/batch-vs-realtime-cost.md`'s batch column.

### 4.2 Latency

Time to first token (TTFT) and total response time. Minimized by:

- **Small batch sizes.** A request that runs alone reaches first-token faster.
- **Speculative decoding.** A small "draft" model predicts several tokens; the large model verifies them in parallel. Reduces wall-clock per token.
- **Quantization.** 8-bit or 4-bit quantized weights load and execute faster (with quality trade-off).
- **Sufficient GPU memory.** A model that fits comfortably leaves bandwidth for the request.
- **Prefix caching.** Reusable system-prompt prefix means the request only generates the new tokens, not the system prompt.

Workloads optimized for latency: chat, real-time agentic loops, interactive Care Coordinator turns.

### 4.3 Memory

VRAM is the binding constraint on most inference deployments. Memory is consumed by:

- **Weights.** A 70B model in fp16 needs ~140 GB; in 4-bit needs ~35 GB.
- **KV cache.** Each concurrent sequence consumes memory proportional to its context length. A 70B model with 8K context and 32 concurrent sequences may need 30+ GB of KV cache.
- **Activations.** Transient during forward pass.
- **Working memory.** Tokenizer state, scheduler state, request buffers.

Memory is what determines max concurrent batch size. Insufficient memory → smaller batches → lower throughput.

### 4.4 The triangle

Increase throughput → larger batches → higher per-request latency *and* more KV-cache memory.
Reduce latency → smaller batches → less throughput.
Reduce memory → quantize → potential quality regression and slower inference.

No deployment optimizes all three. Pick the one that matches the workload's user-visible goal; sacrifice the other two.

### 4.5 The choice for Care Coordinator-shaped workloads

Care Coordinator is interactive. Latency dominates: TTFT is what a user perceives. Throughput per-GPU is secondary, but cost-per-conversation is meaningful. Memory must be sized to handle the long system prompt and the chat history at p95 conversation length.

Optimization order: latency first, memory second (to fit the prompt + history), throughput last. This translates to: smaller batch sizes, aggressive prefix caching, speculative decoding if the runtime supports it, and quantization only after quality is verified.

---

## 5. Deployment shape: dedicated vs shared multiplexed

A self-hosted runtime serves one model per process, by default. The shape decision is how many processes (and instances) host which models.

### 5.1 Dedicated per-model

Each model has its own process and instances. The 70B clinical-knowledge model lives on its own GPU pool; the embedding model lives on a different pool; the reranker on a third.

Strengths:

- Clean isolation. A surge on one model does not affect another.
- Per-model autoscaling.
- Per-model capacity planning is straightforward.
- Per-model failure containment (an OOM on the embedding model does not take down clinical-knowledge).

Weaknesses:

- Under-utilization is common. If the reranker only sees 10% of traffic at peak but you cannot share its GPU, you waste 90% of that GPU.
- Cost grows linearly with model count.

When to use: when each model has steady, predictable, substantial traffic, and the team wants strong isolation. The default for production systems serving few large models.

### 5.2 Shared multiplexed

One runtime process hosts multiple models, swapping them in and out of GPU memory as requests come in. Or one runtime serves multiple model versions side-by-side (continuous batching across models that fit in memory simultaneously).

Strengths:

- Higher GPU utilization across a portfolio of models with uneven traffic.
- Cost-efficient for medium / low-traffic models that would otherwise sit idle.

Weaknesses:

- Cold-swap latency. Swapping a model into GPU memory takes seconds; first request after a swap is slow.
- Head-of-line blocking. A request for model A may wait behind a request for model B.
- More complex routing.
- Harder to reason about capacity per-model.

When to use: when serving a portfolio of models with uneven traffic; when economic constraints rule out dedicated instances for every model.

### 5.3 The hybrid pattern

A common pattern: dedicated for high-volume models, shared for the long tail.

For Care Coordinator:

- Dedicated pool for the clinical-knowledge model (heavy traffic, latency-critical).
- Dedicated pool for the embedding model (steady traffic, batched).
- Shared pool for: drafting model, classifier, query-rewriter, a few experimental fine-tunes (uneven traffic, mostly small models).

The hybrid pattern matches resource shape to traffic shape.

### 5.4 Multi-tenancy considerations

If different tenants get different fine-tuned models, dedicated-per-model becomes dedicated-per-tenant-per-model, which scales poorly. The mitigation:

- LoRA-based fine-tunes share a base model; the runtime swaps LoRA adapters cheaply (milliseconds, not seconds). One base + many adapters supports per-tenant fine-tunes on one shared GPU.
- This pattern is the basis of [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-fine-tuning.md)'s implementation guidance.

---

## 6. Autoscaling for AI workloads

Autoscaling LLM inference is meaningfully different from autoscaling web services.

### 6.1 Why it is different

- **Cold-start time is measured in minutes, not seconds.** A new GPU instance needs to be provisioned, the model image pulled, the weights loaded into VRAM, the runtime warmed. 60–300 seconds end-to-end is typical.
- **Reactive autoscaling does not work.** By the time the metric crosses the threshold, the spike has already harmed users for 2–5 minutes.
- **GPU instance cost is meaningful.** A scale-up event is a real cost decision, not a cheap-and-reversible action.
- **GPU instance availability is not guaranteed.** Cloud GPU capacity is often constrained; a request for 4 more H100s may not be immediately fillable.

### 6.2 The scaling patterns

**Pre-warmed pool.** Always run N+M instances where N covers steady traffic and M is a buffer for surges. The buffer absorbs the cold-start delay. Cost is the buffer.

**Predictive autoscaling.** Scale based on forecasted load, not current load. Anchor to known patterns (mornings have higher traffic than evenings; Mondays are busier than Sundays).

**Scheduled scaling.** Scale up before known peak windows. For Care Coordinator, the morning case-conferencing window is predictable; pre-scale for it at 7 AM local.

**Slow scale-down, fast scale-up.** When scaling down, do so gradually and only after sustained low load. When scaling up, do so aggressively at the first signal — the cost of a hot instance for 10 minutes is far less than the cost of a queue building up.

### 6.3 The capacity signal

Scale on the right metric:

- **Request queue depth.** When requests start queuing, capacity is short. The fastest signal, the earliest to react to.
- **GPU utilization.** When sustained > 75%, capacity is constrained. A reactive signal; lags behind queue depth.
- **TTFT p95.** When p95 first-token latency rises, the system is degrading. The latest to react; by the time TTFT moves, users are already affected.

Compose: queue depth as primary signal, GPU utilization as secondary, TTFT p95 as a confirming check.

### 6.4 GPU instance lifecycle

- **Provisioning.** Request a GPU instance. 30–60s typical on major clouds; can be longer if capacity is constrained.
- **Image pull.** Pull the runtime image. 30–90s for a vLLM image with all dependencies.
- **Weight load.** Load weights from a shared volume or object store. 30–120s for a 70B model depending on quantization and bandwidth.
- **Warm-up.** Run a few dummy requests to compile kernels and prime caches. 10–30s.

Total: 100–300s. Plan capacity assuming a hot instance is 2–5 minutes away from any cold instance.

---

## 7. The GPU choice

The GPU choice has cost and capability implications. Common options in 2026:

### 7.1 H100 (NVIDIA)

The workhorse for serious inference. 80 GB HBM3 per card. Strong FP8 support.

Fit: 70B-class models at high quality. Multiple smaller models per card. Production-grade throughput.

Rough rental: $2.50–4.00/hour on major clouds, more in constrained regions.

### 7.2 H200 (NVIDIA)

H100 successor with 141 GB HBM3e. The memory increase is meaningful for long-context workloads and KV-cache-bound deployments.

Fit: long-context inference, very-large-model serving, multi-tenant LoRA serving.

Rough rental: 20–40% premium over H100.

### 7.3 A100 (NVIDIA)

The previous generation. 40 GB or 80 GB HBM2e variants. Widely available.

Fit: 13B / 34B models, embedding workloads, batch jobs. Cost-effective for non-frontier workloads.

Rough rental: $1.20–2.50/hour.

### 7.4 L40S (NVIDIA)

Inference-targeted card. 48 GB GDDR6. Lower cost than H100, less raw compute, less memory bandwidth.

Fit: medium-model inference (7B–13B), embedding workloads, cost-sensitive deployments where peak throughput is not the constraint.

### 7.5 Alternative vendors

AMD MI300X, AWS Trainium/Inferentia, Google TPU. Each has trade-offs. As of 2026, NVIDIA remains the default for ease-of-deployment because most serving stacks are CUDA-first. The alternatives are viable for teams willing to invest in the integration work for the price-performance gains.

### 7.6 Single-card vs multi-card serving

- **Single-card.** Model fits on one GPU. Lowest complexity, lowest latency, easiest scaling.
- **Tensor-parallel.** Model split across multiple GPUs in one box, all working on the same request. Required for very large models (175B+ unquantized).
- **Pipeline-parallel.** Different layers on different GPUs in a pipeline. Less common in modern LLM serving.

For most production deployments: prefer single-card. Choose a GPU with enough memory to fit the model + KV cache for target batch size, even if it costs more per card.

---

## 8. Operational cost

The cost of self-hosting is not just GPU rental. It is GPU rental *plus* the operational platform investment.

### 8.1 Headcount

A production inference platform needs:

- **An inference-platform engineer.** Lives in the serving stack: configures vLLM/TGI, tunes batch sizes, debugs OOMs, handles model deployments. 1 FTE minimum; 2 FTE for redundancy.
- **A platform SRE.** Monitors capacity, manages autoscaling, runs incident response. Shared with broader SRE.
- **A model-platform engineer.** Bridges between the model lifecycle (registry, promotion) and the runtime (deployment, rollback). Often the same as the inference engineer at small scale.

Floor: 2 FTE dedicated to the inference platform.

### 8.2 GPU spend

Cost = (instances at steady-state) × (hours per month) × (hourly rate) + (buffer pool cost) + (autoscale event cost).

For Meridian's clinical-knowledge model: 4× H100 instances at $3.00/hour, 24/7 = $8,640/month per instance × 4 = $34,560/month steady-state. Plus a 50% buffer pool = $51,840/month. Plus observability, log shipping, etc.

The cost scales linearly with concurrent capacity. The break-even vs vendor APIs depends on the workload's per-token cost on the API and the runtime's utilization.

### 8.3 Observability investment

The runtime needs:

- Per-request tracing (which model version served which request, with what latency, at what cost).
- GPU metrics (utilization, memory pressure, temperature).
- Queue depth and batch size telemetry.
- Per-model error rates.

The investment is comparable to running any other production stateful service. See [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) for the trace shape.

### 8.4 The total cost vs vendor API

A reasonable benchmark for a Care-Coordinator-scale workload:

- Vendor API for the equivalent model: $0.08–0.15 per conversation.
- Self-hosted on H100 at 70% utilization: $0.05–0.10 per conversation, *plus* the operational and engineering cost.

At small scale (< 1B tokens/month), vendor APIs win. At large scale (> 10B tokens/month), self-hosting wins if and only if the platform investment is real.

### 8.5 The honest break-even

Most teams that ask "can we save money by self-hosting" underestimate the platform investment. The break-even where self-hosting saves real money is typically much higher than expected — often 5–10× the workload size where it first looks attractive on raw GPU pricing.

---

## 9. Integration with model registry and CI/CD

Self-hosted models still go through the lifecycle.

### 9.1 Registry integration

Every self-hosted model is registered in [model-registry.md](./model-registry.md), with the same metadata as vendor-hosted models:

- Version, owner, allowed contexts, deprecation date.
- Runtime config (which serving stack, which GPU, which quantization).
- Weight location (S3 path, internal registry path).

The registry is the source of truth. The runtime configuration is derived from it.

### 9.2 Deployment via CI/CD

Self-hosted model deployment goes through the same pipeline as vendor-hosted model promotion:

- Eval gate ([eval-gate-design.md](../cicd-and-eval-gates/)) on the candidate weights.
- Canary ([canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md)) on a small percentage of traffic to a new instance.
- Promotion when the canary clears.

The mechanics differ (new GPU instances rather than a new API call target), but the discipline is the same.

### 9.3 Rollback

Self-hosted rollback is rolling back the weight version on the runtime — pinned per [rollback-procedures.md](./rollback-procedures.md). The runtime has both the old and new weight versions available; rollback is flipping the routing layer's target.

### 9.4 Multi-region

For data residency, the model is deployed per region. Each region has its own registry binding, its own runtime, its own GPU pool. The application layer routes by tenant residency to the correct region.

---

## 10. Worked Meridian example: deploying clinical-knowledge to EU

Meridian has a contract with a Spanish hospital network that requires PHI to remain in EU jurisdiction. The vendor's BAA covers HIPAA but not the Spanish data-residency clauses. The clinical-knowledge model must be self-hosted in Frankfurt.

### 10.1 The decision

The architecture team's analysis (per [model-strategy/build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md)):

- Volume: ~500M tokens/month for the EU tenants.
- Data residency: hard requirement.
- Latency: acceptable in 1–2 second range; not latency-critical.
- Cost: vendor API would cost ~$50K/month at this volume.

Self-hosting decision: yes, on data residency. The cost is secondary; the data residency is non-negotiable.

### 10.2 The deployment shape

- **Serving stack:** vLLM. Default choice; team has experience.
- **Model:** Llama-3-70B-Instruct, the team's chosen open-weight base. Distilled from the vendor's frontier model on production traffic samples (per [distillation-operations.md](./distillation-operations.md)) for the EU tenants' workload.
- **GPU:** 4× H100 dedicated pool in eu-central-1 (Frankfurt).
- **Quantization:** AWQ 4-bit. Quality eval showed minimal regression; allows 2× more concurrent sequences per GPU.
- **Batch size:** 16 concurrent sequences per GPU. Tuned for the latency requirement; could push higher but the latency budget closes.
- **Autoscaling:** Steady-state 4 instances; buffer 2 more; scheduled scale-up to 8 during the EU morning peak (8–10 AM local).

### 10.3 The CI/CD wiring

- New weight versions pushed to the model registry with a region-specific binding (`care_coordinator_clinical_knowledge_eu: llama-3-70b-care-coordinator-distilled-v1.4.2`).
- Deployment pipeline runs eval gate against the EU-specific golden set (held-out cases from EU tenants, anonymized).
- Canary at 5% of EU traffic for 24 hours; promotion at 24 hours if metrics clean.
- Rollback path: previous weight version stays on disk for 14 days; rollback is a config flip.

### 10.4 The capacity model

- p95 conversations per minute (EU peak): 480.
- p95 turns per conversation: 7.
- p95 tokens per turn: 800 in + 600 out.
- Per-GPU throughput at the chosen batch size: ~10 concurrent sequences sustained.
- Required instances at peak: 480/min × 7 turns / 60 = 56 concurrent turn/sec, with overhead → ~6 instances. Pre-scaled to 8.

### 10.5 The cost

- 4 base H100s × $3/hr × 720 hr/month = $8,640.
- 2 buffer H100s × $3/hr × 720 hr × 70% (often idle but charged) = $4,300.
- 2 scheduled peak H100s × $3/hr × 60 hr/month = $360.
- Observability, networking, weight-storage: ~$1,200.
- Total: ~$14,500/month.

vs the $50K/month vendor cost. Real savings, but the operational cost (1.5 EU-region-focused FTE plus shared SRE) brings the total to roughly $50–60K once the platform investment is loaded — break-even on cost, win on data residency.

### 10.6 Findings closed

- **ARCH-CARE-056** (EU residency requirement was being met by routing to a vendor's EU endpoint, which did not satisfy the Spanish hospital's data-residency clauses).
- **ARCH-CARE-057** (no self-hosted inference platform; the EU need had been blocking onboarding of the Spanish customer).
- **ARCH-CARE-058** (the team had not done a build-vs-buy crossover for any workload).
- **ARCH-CARE-059** (no documented runtime selection rationale for any model).

---

## 11. Anti-patterns

### 11.1 The "we'll just run vLLM" prototype-in-production

The team stands up a single vLLM instance behind an nginx, calls it production, and runs into OOMs, no autoscaling, no rollback, no observability. The team's PagerDuty rotation gets paged at every minor issue.

The fix: the platform investment is the platform investment. A serving stack is one ingredient; production is the whole meal.

### 11.2 The cost-only build-vs-buy

The team compares vendor per-token pricing to GPU rental and concludes self-hosting saves money. They have not accounted for the FTE, the buffer pool, the observability, or the operational complexity. The "savings" evaporates within six months.

The fix: load the operational cost into the comparison. The build-vs-buy doc has the framework.

### 11.3 The under-provisioned cold-start

The team scales reactively. When a traffic spike hits, the autoscaler tries to add an instance; cold-start takes 4 minutes; users see degraded service for the entire window. The team blames the autoscaler when the problem is the architecture.

The fix: pre-warmed pool plus predictive scaling. Reactive autoscaling alone does not work for AI inference.

### 11.4 The dedicated-per-model overspend

The team gives each of 15 models its own GPU pool. Most pools sit at 5% utilization. The bill is enormous. The team cannot justify it to finance.

The fix: dedicated for the few high-volume models; shared multiplexed for the long tail. LoRA-based multi-tenant fine-tuning for per-tenant variants.

### 11.5 The quantization gamble

The team quantizes to 4-bit to halve memory cost without evaluating quality. Quality regresses, but no one notices for a month because the eval is offline-only and the regression is on a subset of queries the offline eval does not cover.

The fix: quantization is a model change. Run eval + canary + A/B (per [ab-model-testing.md](./ab-model-testing.md)) before promoting quantized weights.

### 11.6 The "we have one engineer" platform

A single engineer carries the entire inference platform. They have all the tribal knowledge. They go on PTO; an incident lands; nobody else can debug it.

The fix: two engineers, shared on-call, shared documentation. Tribal knowledge is a single point of failure.

### 11.7 The latency-versus-throughput confusion

The team tunes the runtime for maximum throughput (large batches, deep queues) and is surprised that interactive features feel sluggish. The two dimensions trade off.

The fix: tune to the workload's user-visible goal. Throughput for batch jobs; latency for interactive. Separate pools if both shapes coexist.

### 11.8 The forgotten weight refresh

A model weight version is deployed; the team moves on. Months later, the upstream base model has a known issue that has been fixed in a newer version. The team's deployment has been silently behind. No process exists to evaluate and adopt the newer base.

The fix: model versions follow the deprecation playbook ([model-deprecation-playbook.md](./model-deprecation-playbook.md)). Self-hosted models age out on the same schedule.

---

## 12. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| ML-RT-001 | Self-host decision made on cost alone; operational cost not loaded | High | Apply [build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md) framework; load FTE and platform cost into comparison | Architecture + AI Platform |
| ML-RT-002 | No documented serving-stack rationale | High | Document the choice per workload; reference §3.5 decision table | AI Platform |
| ML-RT-003 | Single inference engineer on-call; no redundancy | High | Build to 2 FTE minimum; shared on-call rotation | AI Platform |
| ML-RT-004 | Reactive autoscaling only; cold-start exceeds spike duration | High | Add pre-warmed buffer; introduce scheduled or predictive scale-up | AI Platform + SRE |
| ML-RT-005 | Per-request latency dashboard absent; team troubleshoots blind | High | Add TTFT, queue depth, batch size metrics; integrate with [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) | AI Platform |
| ML-RT-006 | Quantization shipped without eval | High | Treat quantization as a model change; run eval + canary + A/B before promotion | AI Platform + Eval Eng |
| ML-RT-007 | Dedicated-per-model for low-traffic models; utilization < 20% | Medium | Move long-tail models to shared multiplexed serving or LoRA-on-shared-base | AI Platform |
| ML-RT-008 | No regional residency strategy; vendor endpoints used in restricted jurisdictions | High | Self-host in restricted regions; tenant routing by residency | Architecture + AI Platform |
| ML-RT-009 | GPU choice undocumented; multi-cloud inconsistency | Medium | Document GPU choice per workload per region; standardize where possible | AI Platform |
| ML-RT-010 | Rollback path for self-hosted models untested | High | Test rollback weekly in staging; verify weight-version flip path works | AI Platform + SRE |
| ML-RT-011 | Self-hosted models not registered in model registry | High | All self-hosted models registered with same metadata as vendor-hosted | AI Platform + Architecture |
| ML-RT-012 | Weight version aging without refresh cadence | Medium | Apply [model-deprecation-playbook.md](./model-deprecation-playbook.md) to self-hosted models; quarterly base-model evaluation | AI Platform |
| ML-RT-013 | Cold-start time unmeasured; capacity planning incorrect | Medium | Measure cold-start per runtime/model combination; bake into capacity model | AI Platform + SRE |
| ML-RT-014 | Multi-tenant fine-tunes deployed as dedicated instances per tenant; cost explosion | High | LoRA-on-shared-base for per-tenant variants per [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-fine-tuning.md) | AI Platform + Architecture |
| ML-RT-015 | Capacity model assumes vendor-API characteristics for self-hosted runtime | Medium | Build self-hosted capacity model from GPU throughput, batch size, and request shape | SRE + AI Platform |
| ML-RT-016 | Prefix-caching not used; system prompt re-tokenized every request | Medium | Enable prefix caching (RadixAttention or runtime equivalent); measure throughput gain | AI Platform |
| ML-RT-017 | Speculative decoding not evaluated; interactive latency above budget | Low | Pilot speculative decoding for the chat path; verify quality unchanged | AI Platform |
| ML-RT-018 | No tested degraded-mode for runtime failure; failover only to vendor API at full price | Medium | Define degraded-mode per [reliability-engineering/degraded-mode-design.md](../reliability-engineering/degraded-mode-design.md); include cost-controlled fallback | AI Platform + SRE |

---

## 13. Adoption checklist

- [ ] Self-hosting decision made via [build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md) framework with operational cost loaded.
- [ ] Serving-stack chosen and rationale documented per workload (vLLM / TGI / SGLang / Triton).
- [ ] At least 2 FTE on the inference platform with shared on-call.
- [ ] Pre-warmed buffer pool plus scheduled or predictive scale-up; reactive autoscaling alone not relied upon.
- [ ] Per-request observability live: TTFT, queue depth, batch size, GPU utilization, error rates.
- [ ] Quantization treated as a model change; eval and A/B run before promotion.
- [ ] Deployment shape (dedicated vs shared vs hybrid) chosen per workload and documented.
- [ ] All self-hosted models in the model registry with full metadata including runtime config.
- [ ] Multi-region deployment for residency-constrained workloads.
- [ ] Rollback path tested weekly in staging.
- [ ] Capacity model built from runtime characteristics, not vendor-API extrapolation.
- [ ] Prefix caching enabled where the workload shape supports it.
- [ ] Cold-start time measured per runtime / model combination.
- [ ] Multi-tenant fine-tunes use LoRA-on-shared-base pattern.
- [ ] Degraded-mode failover defined; runtime failures route to a tested fallback.

---

## 14. References

**Internal:**

- [model-registry.md](./model-registry.md) — the registry that lists self-hosted models alongside vendor-hosted.
- [model-promotion.md](./model-promotion.md) — the deployment pipeline self-hosted models go through.
- [rollback-procedures.md](./rollback-procedures.md) — the rollback path for self-hosted weights.
- [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md) — the canary that gates the self-hosted promotion.
- [ab-model-testing.md](./ab-model-testing.md) — the production-mix comparison after the canary.
- [distillation-operations.md](./distillation-operations.md) — where the self-hosted fine-tune came from.
- [fine-tuning-operations.md](./fine-tuning-operations.md) — the production fine-tune lifecycle.
- [model-deprecation-playbook.md](./model-deprecation-playbook.md) — the deprecation cadence for self-hosted base models.
- [reliability-engineering/capacity-planning.md](../reliability-engineering/capacity-planning.md) — capacity modeling for AI services.
- [reliability-engineering/multi-provider-failover.md](../reliability-engineering/multi-provider-failover.md) — fallback to vendor API on self-hosted failure.
- [reliability-engineering/degraded-mode-design.md](../reliability-engineering/degraded-mode-design.md) — degraded-mode patterns.
- [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — trace shape for inference calls.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — attributing the GPU spend.

**Cross-repo (architecture sibling):**

- [model-strategy/build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md) — the decision framework that precedes self-hosting.
- [model-strategy/frontier-vs-open-weights-vs-fine-tune.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/frontier-vs-open-weights-vs-fine-tune.md) — the model-choice context for what to self-host.
- [cost-and-performance-architecture/gpu-strategy-for-self-hosted.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/cost-and-performance-architecture/gpu-strategy-for-self-hosted.md) — architecture-side GPU strategy.
- [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-fine-tuning.md) — LoRA pattern for multi-tenant deployments.
- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
