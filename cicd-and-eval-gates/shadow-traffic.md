# Shadow Traffic

> **Audience.** Engineers evaluating whether a candidate AI version should be canaried (user-impacting traffic) or shadowed (no user impact). Platform leads who have asked "can we test this in production without risking a single user?" and need the structural pattern. **Scope.** The *engineering* pattern for shadow traffic: running new prompts / models / pipelines in parallel against production traffic without serving the new outputs to users; the offline-comparison discipline; the cost trade-off (shadow doubles the AI bill on the shadowed traffic); when shadow is the right tool vs canary; the limits of shadow (what cannot be measured in shadow). Pair with [canary-rollouts.md](./canary-rollouts.md) (the user-impacting alternative). Cross-link to [model-lifecycle/canary-and-shadow-rollout.md](../model-lifecycle/canary-and-shadow-rollout.md) (the model-level mechanic) and [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) (the offline scoring that compares shadow vs production). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Shadow traffic is canary's quieter cousin. Canary routes a fraction of users to the new version; shadow routes *no* user to the new version but runs the new version *in parallel* against the same requests, capturing its output for offline comparison. The user sees the production version's output every time; the shadow version's output is collected, scored, and analyzed without ever reaching a user.

The reason shadow exists is that canary's user-impacting nature is not always acceptable. For high-stakes clinical features, a 1% canary still means 1% of users (or conversations) see an experimental version's output. For regulated decisions, a single user receiving a wrong answer from a candidate model can be a reportable event. For latency-critical paths, even 1% of users seeing a slower version may breach an SLO. In all of these, shadow is the answer: production behaves exactly as before, while the team gathers data on the candidate version.

The downside of shadow is real: every shadowed request runs *twice* — once for production, once for shadow — and the team pays for both. At meaningful traffic volume, shadow doubles the AI bill on the shadowed traffic. Some signals are also unmeasurable in shadow: user thumbs-up requires showing the user the response; conversation continuation requires the user to engage with the response; downstream user behavior (did they click? did they convert? did they escalate?) is invisible in shadow.

The pattern in this document is when to reach for shadow, how to run it, what signals it produces, and what its limits are. The honest framing: shadow is more expensive than canary on cost but cheaper on user-impact risk. Most teams use both — shadow for the riskiest changes (model swaps, fine-tunes for clinical workloads); canary for everything else. The selection is a per-change decision.

This document is opinionated about four things:

1. **Shadow is for changes where user-impact risk is the binding constraint.** When the team cannot risk even 1% of users seeing the candidate's output. Not the default; selected when the change warrants it.
2. **Shadow doubles cost on shadowed traffic. Budget it.** The "free signal" framing is wrong. Shadow has a real bill.
3. **Some signals are not measurable in shadow.** User thumbs-up, conversation continuation, downstream behavior. Plan around the absence of these.
4. **Shadow ends with a canary, not with promotion.** Shadow validates safety; canary validates production-mix behavior in user-impacting traffic. Both, in sequence, for high-risk changes.

Structure: (2) when to use shadow vs canary; (3) the shadow mechanic; (4) what shadow measures well; (5) what shadow cannot measure; (6) the cost trade-off; (7) shadow duration and stopping rule; (8) the shadow-to-canary handoff; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. When to use shadow vs canary

Both patterns ship a candidate version. The distinction is who pays the user-impact cost.

### 2.1 Use canary when

- The change is medium- to low-risk (sub-version model refresh, prompt tweak, retrieval-corpus update).
- User-impact at 1% traffic is tolerable.
- Signals that require user interaction (thumbs-up, conversation flow) are the load-bearing metrics.
- Cost is constrained (canary at 1% costs only 1% extra).

### 2.2 Use shadow when

- The change is high-risk: model swap, fine-tune deployment for a clinical / regulated workload, structural pipeline change.
- User-impact at 1% traffic is not tolerable (regulatory, contractual, or product risk).
- The team needs to verify *safety* (refusal behavior, hallucination rate, schema compliance) before exposing any users.
- The traffic volume is low enough that user-impact at 1% would not produce statistical signal anyway; shadow at 100% produces more signal than canary at 1% for low-volume features.
- Quality and reliability signals are sufficient; user-interaction signals are nice-to-have but not load-bearing.

### 2.3 Use both (shadow then canary) when

- The change is high-risk *and* user-interaction signals matter.
- Shadow runs first to validate safety; canary follows to measure user-impact metrics.
- This is the gold-standard pattern for major model upgrades on clinical workloads.

### 2.4 Use neither when

- The change has no user-visible behavior at all (a refactor that doesn't change outputs).
- The change is an emergency hotfix where speed beats validation rigor.
- The change cannot be canaried or shadowed cleanly (schema-breaking change requiring synchronized rollout).

### 2.5 The decision table

| Risk | User-interaction signal load-bearing? | Pattern |
|---|---|---|
| Low | Yes | Canary |
| Low | No | Canary or eval-only |
| Medium | Yes | Canary |
| Medium | No | Canary or shadow |
| High | Yes | Shadow → Canary |
| High | No | Shadow |
| Refactor (no output change) | n/a | Eval-only |

---

## 3. The shadow mechanic

How shadow actually runs.

### 3.1 The shape

For every production request:

1. The request hits the routing layer.
2. The routing layer dispatches the request to the production version (whose output goes to the user).
3. The routing layer *also* dispatches the request to the shadow version (whose output goes to a collection store).
4. The user receives the production version's output.
5. Offline, the shadow output and production output are compared (scored by judge, diffed, aggregated into metrics).

### 3.2 The implementations

**Fire-and-forget dispatch.** The shadow call is dispatched asynchronously; the production response goes back to the user immediately. The shadow call completes in the background.

- Pros: zero user-visible latency from shadow.
- Cons: shadow can fall behind on backpressure; coordination is loose.

**Synchronous-with-cap dispatch.** The shadow call is awaited up to a cap (e.g., 500ms); if it doesn't complete in time, the shadow result for that request is recorded as "timeout" and the production response proceeds.

- Pros: shadow doesn't fall behind.
- Cons: small risk of adding latency if the cap is set too high.

**Replay-from-log dispatch.** Production requests are logged; a separate offline job replays them against the shadow version later.

- Pros: zero impact on production hot path.
- Cons: the shadow runs on stale data; signals are delayed by the replay lag.

For most teams: synchronous-with-cap is the default. Fire-and-forget is acceptable for non-critical features. Replay-from-log is useful for very-large shadow runs where production-time impact must be zero.

### 3.3 The collection store

Shadow outputs go to a dedicated store:

- Per-request: the request input, the production output, the shadow output, the latency / cost / token-count of each, the timestamp, the trace id.
- Retention: typically the duration of the shadow run plus a comparison window (30 days).
- Privacy: shadow outputs may contain the same sensitive data as production outputs; the store has the same access controls.

### 3.4 The comparison job

Offline, periodically (or at the end of the shadow window):

- The judge ([eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md)) scores both production and shadow outputs.
- Diff metrics aggregate: how often did shadow refuse where production answered? How often did shadow disagree materially? Schema-compliance per arm. Cost per arm. Latency per arm.
- A report is generated, attached to the candidate release.

### 3.5 What gets routed to shadow

By default, all production requests for the shadowed feature go to shadow too. Alternatives:

- **Sampled shadow:** 10% of production goes to shadow, the rest is production-only. Saves cost; reduces signal volume.
- **Stratified shadow:** specific request types are shadowed; others are not. Useful when only a subset of behavior is being tested.
- **Tenant-restricted shadow:** only certain tenants' traffic is shadowed. Required when shadowing on tenants that have not consented to it, or when the candidate involves tenant-specific behavior.

The default is full shadow. The sampling alternatives are cost optimizations.

---

## 4. What shadow measures well

Shadow is the right tool for these signals.

### 4.1 Output-level quality

The judge scores shadow outputs on the same rubric as production outputs. The delta is the quality signal.

- Live-judge quality on shadow vs production.
- Stratified by request type, by tenant, by query length.
- Confidence intervals reported.

This is the headline signal. Shadow's quality measurement is as reliable as canary's, with the benefit of 100% of traffic shadowed (vs canary's small percentage).

### 4.2 Schema compliance

For structured-output features, the shadow's output is parsed against the schema. Failures are counted.

- Schema-compliance rate on shadow vs production.
- Specific schema violation patterns (which fields fail validation, in what ways).

### 4.3 Refusal behavior

Did the shadow refuse where production answered? Did the shadow answer where production refused?

- Refusal rate on shadow vs production.
- Refusal-pattern analysis: which query patterns trigger refusal in shadow that didn't in production?

### 4.4 Cost and latency

The shadow records cost and latency per request, even though those values don't affect users.

- Cost per request on shadow vs production.
- p50 / p95 / p99 latency.
- Token counts per request.

Latency from shadow is technically what the user *would have seen* had the shadow been serving; this is a useful planning signal.

### 4.5 Output diffs

Where shadow and production produce different outputs (on the same input), the diff is captured:

- For freeform text: a diff metric (Levenshtein, embedding distance, judge-comparison score).
- For structured output: field-by-field difference.
- For numeric or boolean output: simple agreement rate.

Diff distributions reveal the *shape* of the candidate's behavior change, not just the headline pass-rate.

### 4.6 Tool-call behavior

For agentic features:

- Tool-call selection: did shadow call the same tool as production?
- Tool-call arguments: same arguments?
- Tool-call success rate (the shadow's calls go to real tools, with real side effects — see §10.6 for the caveat).

### 4.7 Hallucination indicators

If the team has a hallucination check (golden facts the model should not contradict, citation requirements, etc.), shadow lets you measure them at 100% of traffic.

- Hallucination rate on shadow vs production.
- Citation accuracy.

---

## 5. What shadow cannot measure

Shadow's blind spots are real. Plan for them.

### 5.1 User-interaction signals

The user never sees the shadow output. Therefore the team cannot measure:

- **Thumbs-up / thumbs-down rate.** No user is rating the shadow's response.
- **Conversation continuation.** Did the user keep talking after the response? The user is reading the production response; their decision to continue is about production, not shadow.
- **User-text follow-up.** Did the user's next message indicate satisfaction or confusion? Same problem.
- **Click-through / conversion.** For features driving user action, the action is taken on production output.

These signals require canary. Shadow does not substitute.

### 5.2 Multi-turn behavior

For chat / multi-turn features, the shadow's turn-1 output never reaches the user, so the shadow cannot "see" turn 2 of the same conversation (unless the routing layer replays the conversation against the shadow, which has its own caveats — see §10.4).

In practice:

- Shadow per-turn is well-measured (turn-1 quality compared per-turn).
- Shadow as a whole conversation is approximate (the shadow doesn't truly experience the conversation).

For features where multi-turn behavior is the load-bearing property (a coordinator carrying state across turns), shadow's signal is partial.

### 5.3 Tool-call side effects on real systems

For agentic features where tools have real-world side effects (sending an email, creating a ticket, modifying a database):

- Shadow that *executes* tools will produce duplicate side effects (the user gets two emails: one from production, one from shadow).
- Shadow that *does not* execute tools loses tool-result data; the shadow's downstream reasoning is based on missing or stub tool results.

Neither is clean. Most teams handle this by:

- Read-only tools: shadow executes them; cheap, no duplicates.
- Side-effecting tools: shadow uses stub tools or recorded production results; loses some signal.

The choice is per-tool and per-feature.

### 5.4 Backend system load

A shadow that runs at full traffic doubles backend load on:

- The model provider (twice as many calls).
- Embedding models (twice as many embeddings).
- Vector store reads (twice as many retrievals).
- Tool calls (per §5.3).

The shadow's measurement of the candidate's *isolated* behavior is good; the shadow's measurement of how the candidate behaves *under production load* is not, because production is already running its own load.

### 5.5 Tail latency under saturation

If the production version is running at saturation (p99 latency tail), the shadow won't see that saturation pattern unless the shadow's own infrastructure is similarly saturated. Shadow's latency profile is "candidate under unstressed conditions"; production's is "production under realistic conditions."

For latency-critical workloads, canary is necessary to verify behavior under real load.

---

## 6. The cost trade-off

Shadow's cost is not "minor." Plan for it.

### 6.1 The cost shape

For a feature that serves N requests per day at a per-request cost of $C:

- Production-only cost: N × C per day.
- Full-shadow cost: 2 × N × C per day (production + shadow).
- 10% sampled shadow: 1.1 × N × C per day.

The shadow's marginal cost is roughly equal to the production cost; full shadow doubles the bill.

### 6.2 Cost-aware shadow

Strategies to reduce shadow cost:

- **Sample.** Shadow 10% of production; signal volume is 10x lower, cost is 1.1x.
- **Stratify.** Shadow only request types where signal is most needed.
- **Limit duration.** Shadow for 48 hours to gather signal, not for 14 days.
- **Use a smaller eval suite during shadow.** If the shadow is being scored by an LLM judge, the judge is itself an LLM call; sample the judge's input.

### 6.3 The cost vs canary

Comparing a 14-day shadow to a 14-day canary at 10%:

- Shadow cost: 1× full production cost for shadow, plus 1× production cost = 2× total.
- Canary cost: 0.1× production cost for canary, plus 0.9× production cost for baseline = 1× total.

The canary is dramatically cheaper. The trade is that the canary is 10% of traffic on the candidate (the canary user is exposed to the candidate) whereas the shadow has 0% user-impact.

For some workloads, the cost differential is worth it (a doubled bill for 14 days is acceptable if it prevents a user-impacting incident on a clinical workload). For others, the canary is the better choice and the candidate's safety is verified through other means (eval, controlled testing).

### 6.4 The shadow-then-canary cost

The combined pattern (shadow validates safety, canary validates user-mix):

- Phase 1: shadow at 100% for 24–48 hours. Cost: 1× × 1.5 days = 1.5× production cost for that period.
- Phase 2: canary at 1% → 10% → 50% → 100% over ~32 hours. Cost: ~1× production cost for the canary period.

The combined cost is roughly 2.5× the production cost for the 3–4 day validation period. For high-risk changes, this is defensible.

### 6.5 The "shadow is free" myth

Some teams assume the shadow is free because it produces signal without affecting users. The signal is valuable; the cost is real. Budget the shadow as a deliberate spend.

---

## 7. Shadow duration and stopping rule

How long the shadow runs.

### 7.1 The minimum duration

- **Statistical minimum.** Enough samples for the headline metrics. For live-judge quality at 0.5-point detection, ~1000–5000 samples per arm; at full shadow this is hours to a day.
- **Calendar minimum.** Cover one full weekly cycle (7 days) for features with weekly traffic patterns. For features without weekly patterns, 24–48 hours.
- **Diurnal minimum.** Cover at least one peak window plus one off-peak.

A practical floor: 48 hours for non-weekly features; 7 days for features with weekly patterns.

### 7.2 The stopping rule

Pre-commit at the start of the shadow:

- Duration target.
- Stopping criteria: the candidate passes if quality delta ≥ X, refusal delta ≤ Y, cost delta ≤ Z, etc.
- Pre-shadow definition of "what we're testing for."

At the end of the duration, the team reads the result. The stopping rule prevents the team from "peeking" daily and stopping when the result looks good, which inflates false positives.

### 7.3 The safety stop

Like canary, shadow has safety stops:

- Egregious refusal-rate spike: stop and investigate.
- Schema-compliance crashes: stop and investigate.
- Cost explosion: stop (don't keep paying for a clearly-broken candidate).

Safety stops are *not* the read of the result; they are emergency conditions that abort the shadow before its planned end.

### 7.4 The duration vs cost trade-off

A longer shadow is more confident but more expensive. The team's call:

- For a frontier-vendor sub-version refresh: 24 hours of shadow is usually plenty.
- For a major model swap or fine-tune: 7 days.
- For a routine prompt tweak: probably canary instead.

---

## 8. The shadow-to-canary handoff

Shadow ends with a canary, not with promotion.

### 8.1 Why shadow doesn't end with promotion

Shadow validates safety (refusal, schema, hallucination, output-quality). It does not validate user-interaction (thumbs-up, conversation flow, downstream behavior). For changes where user-interaction matters, shadow alone is insufficient.

### 8.2 The standard sequence

For high-risk changes:

1. Offline eval gate (per [eval-gate-design.md](./eval-gate-design.md)). Pass.
2. Shadow at 100% of traffic, 24–48+ hours. Pass.
3. Canary at 1% → 10% → 50% → 100%. Pass.
4. Promote.

Each step gates the next. Shadow eliminates user-impact risk; canary covers user-interaction signal and load behavior.

### 8.3 The handoff artifact

Before canary starts, the shadow's results are summarized:

- Quality delta (with confidence intervals).
- Refusal-pattern shifts.
- Cost / latency deltas.
- Any concerns surfaced for the canary to watch.

The canary on-the-hook engineer reviews the shadow result before starting canary. If the shadow showed a concerning pattern, the canary may proceed with tighter criteria or smaller initial percentage.

### 8.4 The "shadow-only" exception

For changes where user-interaction signals are not load-bearing (a refactor that should not change output behavior; a cost-optimization that should be invisible to users), shadow can stand alone:

- Shadow validates that outputs match production.
- No canary needed; promote on shadow alone.

This is a small fraction of changes. Most need canary too.

### 8.5 The "canary-only" exception

For changes where user-impact risk is low (a minor prompt tweak), canary alone is sufficient. Shadow is overkill; the canary's 1% exposure is acceptable.

This is the majority of changes.

---

## 9. Worked Meridian example: shadowing a major clinical-knowledge model swap

The Care Coordinator's clinical-knowledge model has been on Anthropic's Claude Opus for 18 months. The team is evaluating a swap to a fine-tuned Llama-3-70B (distilled from Opus on Care Coordinator's production traffic; see [model-lifecycle/distillation-operations.md](../model-lifecycle/distillation-operations.md)). The motivation: cost reduction at the EU scale where data residency forces self-hosting anyway.

### 9.1 Why shadow first

This is a high-risk change:

- Different model family (Anthropic → Meta).
- Different scale (Opus → 70B).
- Clinical workload; safety regression risk is high.
- Any single wrong answer to a real user could be a clinical near-miss.

Direct canary would expose 1% of clinical conversations to an unproven model. Shadow first; canary only after shadow clears.

### 9.2 The shadow plan

- **Duration:** 7 days. Full weekly cycle.
- **Traffic:** 100% of EU clinical-knowledge calls (the workload being swapped).
- **Routing:** synchronous-with-cap dispatch; 1.5-second cap.
- **Stopping rule:** at 7 days, read the result. Earlier safety stops on egregious refusal-rate or schema-compliance regression.
- **Cost budget:** the EU clinical-knowledge spend is ~$15K/month; 7 days of full shadow doubles spend for that period = ~$3.5K additional. Approved.
- **Signals to gather:** live-judge quality, refusal-pattern delta, schema-compliance delta, hallucination rate, cost per request, latency, output diff distribution.

### 9.3 The infrastructure

- Shadow Llama-3-70B-fine-tuned-v1.4.2 deployed on dedicated H100 pool in eu-central-1 (per [model-lifecycle/runtime-platform.md](../model-lifecycle/runtime-platform.md)).
- Routing layer extended to dispatch every clinical-knowledge call to both production (Opus) and shadow (Llama).
- Shadow outputs land in a dedicated collection store.
- The judge job runs every 6 hours, scoring both arms' outputs.

### 9.4 The run

Day 1: shadow starts. Initial signal flowing.

- Schema compliance: 99.6% (Llama) vs 99.8% (Opus). Within tolerance.
- Refusal rate: 1.4% (Llama) vs 1.1% (Opus). +0.3pp. Watching.
- Live-judge quality: 7.21 (Llama) vs 7.42 (Opus). -0.21 points. Within shadow's noise floor for this judge.

Day 3: midpoint check. Patterns stabilizing.

- Schema compliance: 99.7% (Llama) vs 99.8% (Opus). Acceptable.
- Refusal rate: 1.3% vs 1.1%. +0.2pp. Acceptable.
- Live-judge quality: 7.24 vs 7.42. -0.18 points. Slightly concerning.
- Hallucination indicator (citation accuracy): 94.1% vs 95.8%. -1.7pp. **Watching closely.**
- Cost per request: $0.012 (Llama on self-hosted) vs $0.082 (Opus). -85%. As expected.

Day 7: end of shadow.

- Schema compliance: 99.7% vs 99.8%. Acceptable.
- Refusal rate: 1.3% vs 1.1%. +0.2pp. Slight over-refusal on some edge cases.
- Live-judge quality: 7.25 vs 7.42. -0.17 points. Below ideal but within tolerance.
- Hallucination indicator: 94.0% vs 95.8%. -1.8pp. **Concerning.**
- Output diff: 32% of outputs are materially different (judge-comparison score > threshold). Distribution: mostly Llama answers being shorter and slightly less detailed; some Llama answers omit a citation Opus included.
- Cost per request: $0.012 vs $0.082. -85% as expected.

### 9.5 The decision

The shadow result is mixed. Quality on the rubric is acceptable; safety on schema and refusal is fine; the citation-accuracy regression is the concern. The team has two options:

- **Option A:** proceed to canary with tighter criteria, particularly watching citation behavior. If canary confirms the shadow's signal on real-user-impact metrics, the team accepts the trade for the 85% cost reduction.
- **Option B:** invest in further fine-tuning to close the citation gap before canary. Rerun shadow after fine-tune update.

The team picks Option B: the citation-accuracy regression is too concerning for a clinical workload to accept without further work. Shadow has paid for itself by catching a problem that would have hit users in canary.

### 9.6 The shadow-then-canary path (counterfactual)

In a counterfactual where the shadow had cleared:

- The team would have proceeded to canary.
- Canary at 1% → 10% → 50% → 100% over 32 hours.
- Canary criteria tightened on citation accuracy (the signal shadow flagged).
- Promotion at 100% if canary clean.

### 9.7 The cost accounting

- Shadow cost: $3.5K incremental over 7 days.
- Avoided cost (had Option B not happened and canary detected the citation regression at 10% with rollback): ~$300 in canary cost + the user-impact harm in clinical near-misses.
- Net: shadow's $3.5K bought the early-warning of the citation regression. Worth it.

### 9.8 Findings closed

- **ARCH-CARE-084** (high-risk model swaps deployed direct-to-canary; shadow not considered).
- **ARCH-CARE-085** (no shadow infrastructure; "could we shadow?" was structurally infeasible).
- **ARCH-CARE-086** (citation-accuracy not tracked as a separate metric; quality alone is too coarse for clinical workloads).
- **ARCH-CARE-087** (shadow vs canary selection criteria undocumented; risk-vs-cost framework absent).

---

## 10. Anti-patterns

### 10.1 The "shadow is free" assumption

The team assumes shadow has no cost because no user is affected. Six weeks later, the bill is double for the shadowed feature; finance asks who approved it.

The fix: budget shadow as a deliberate spend. The cost is real; the value is "no user-impact risk." That trade may be worth it; the cost is not zero.

### 10.2 The shadow-then-skip-canary

The shadow clears; the team promotes directly to 100%. User-interaction signals (thumbs-up, conversation flow) were never measured. A user-experience regression slips through.

The fix: shadow validates safety; canary validates user-mix. Both, in sequence, for high-risk changes that involve user-interaction.

### 10.3 The shadow with no judge

The shadow captures outputs but no one scores them. The team reads a hundred examples by hand and declares the candidate "looks good." No statistical signal; no audit trail.

The fix: the judge runs on shadow output the same way it runs on production output. Statistical comparison, not anecdotal.

### 10.4 The conversation-state confusion

For multi-turn features, the shadow's per-turn output never reaches the user, so turn 2 of the conversation in production uses production-turn-1 state; turn 2 in shadow uses shadow-turn-1 state. The two conversations diverge after turn 1; comparison becomes incoherent.

The fix: shadow conversations are reset per-conversation; each conversation is independently shadowed with its own state lineage. Mixed-state comparisons are explicitly avoided.

### 10.5 The side-effect-doubled tools

Shadow executes side-effecting tools. Users get two emails, two tickets, two database updates. Finance / support / the database all flag the duplication.

The fix: side-effecting tools are stubbed for shadow, or read-only equivalents are used, or shadow tool execution is logged but not executed.

### 10.6 The infinite-shadow

The team runs shadow indefinitely "to keep collecting data." Cost accumulates; the team never promotes; the candidate is effectively a permanent shadow. No decision is ever made.

The fix: pre-commit duration and stopping rule. At the end, decide: promote (proceed to canary, then 100%), or discard (the candidate doesn't justify proceeding).

### 10.7 The shadow-as-eval-replacement

The team runs shadow instead of offline eval. The shadow runs on production traffic — which is real-world but not curated. Edge cases, regulatory probes, and adversarial inputs are under-represented.

The fix: shadow complements offline eval; it does not replace. The eval suite has cases the production traffic distribution doesn't cover.

### 10.8 The sample-from-only-one-tenant

Shadow runs on a single tenant's traffic. The signal is representative for that tenant only; the team assumes it generalizes to others. It doesn't, and the cross-tenant regression appears in canary or production.

The fix: shadow stratified across tenants (or at least covering the major segments) when the candidate's behavior may vary by tenant.

---

## 11. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| CICD-SH-001 | Shadow vs canary selection criteria undocumented | High | Document decision per §2.5; risk-vs-cost framework | AI Platform |
| CICD-SH-002 | High-risk changes (model swaps, fine-tunes for clinical) bypass shadow | High | Shadow before canary for high-risk per §8.2 | AI Platform + Product |
| CICD-SH-003 | Shadow infrastructure absent; cannot run shadow even when warranted | High | Build routing-layer support and collection store per §3 | AI Platform |
| CICD-SH-004 | Shadow cost not budgeted; treated as free | Medium | Budget shadow as deliberate spend per §6 | FinOps + AI Platform |
| CICD-SH-005 | Shadow runs without judge; outputs unscored | High | Judge runs on shadow output per §4.1; statistical signal | Eval Eng + AI Platform |
| CICD-SH-006 | Shadow promotes directly to 100%; user-interaction signals never measured | High | Shadow-then-canary for changes involving user-interaction | AI Platform |
| CICD-SH-007 | Shadow side-effects double on tool calls; duplicate emails / tickets / DB writes | High | Side-effecting tools stubbed for shadow per §10.5 | AI Platform + Tool Owners |
| CICD-SH-008 | No duration / stopping rule pre-committed; shadows run indefinitely | Medium | Pre-commit duration and stopping criteria per §7.2 | AI Platform |
| CICD-SH-009 | Shadow runs only on one tenant; cross-tenant signal missing | Medium | Stratified shadow across tenant segments per §10.8 | AI Platform |
| CICD-SH-010 | Conversation-state mixing across shadow / production turns | Medium | Per-conversation state lineage; isolate shadow conversations | AI Platform |
| CICD-SH-011 | Shadow's hallucination / citation signal not captured | Medium | Add citation-accuracy and hallucination-indicator metrics per §4.7 | Eval Eng |
| CICD-SH-012 | Shadow safety stops not defined; cost runaway possible | Medium | Define safety stops per §7.3 | AI Platform + FinOps |
| CICD-SH-013 | Shadow result not formalized into release artifact | Medium | Shadow result attached to candidate release per §8.3 | AI Platform |
| CICD-SH-014 | Shadow used as replacement for offline eval | Medium | Shadow complements eval, doesn't replace per §10.7 | Eval Eng + AI Platform |
| CICD-SH-015 | Shadow latency dispatch un-capped; production hot-path can slow | Medium | Synchronous-with-cap or fire-and-forget per §3.2 | AI Platform |
| CICD-SH-016 | Backend load doubled by shadow during high-traffic windows; saturation risk | Medium | Capacity model includes shadow load; throttle shadow during peak | SRE + AI Platform |
| CICD-SH-017 | Output-diff metric absent; shape-of-change unmeasured | Low | Diff metric per §4.5 | Eval Eng |
| CICD-SH-018 | Shadow data retention exceeds privacy compliance window | High | Retention policy aligned with PII / PHI rules; shadow store gated to same access | Privacy + AI Platform |

---

## 12. Adoption checklist

- [ ] Shadow vs canary selection criteria documented; risk-vs-cost framework in use.
- [ ] Shadow infrastructure exists: routing-layer dispatch, collection store, judge job.
- [ ] Shadow dispatch mode chosen per feature (sync-with-cap default; fire-and-forget for non-critical; replay-from-log for very-large runs).
- [ ] Judge scores shadow output the same way it scores production output.
- [ ] Shadow duration and stopping rule pre-committed per shadow run.
- [ ] Safety stops defined for shadow (refusal-spike, schema-crash, cost-runaway).
- [ ] Shadow cost budgeted; treated as deliberate spend.
- [ ] Side-effecting tools stubbed or read-only-equivalent for shadow.
- [ ] Per-conversation state lineage; shadow conversations isolated from production conversations.
- [ ] Hallucination / citation indicators measured on shadow.
- [ ] Output-diff metric captured per shadow run.
- [ ] Shadow stratified across tenants when cross-tenant behavior matters.
- [ ] Shadow latency dispatch capped to protect production hot-path.
- [ ] Backend load model includes shadow's incremental load.
- [ ] Shadow result formalized; attached to candidate release artifact.
- [ ] Shadow privacy retention aligned with PII / PHI rules.
- [ ] Shadow-to-canary handoff documented; canary engineer reviews shadow result before starting canary.

---

## 13. References

**Internal:**

- [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) — the pipeline this shadow stage sits in.
- [canary-rollouts.md](./canary-rollouts.md) — the canary that typically follows shadow.
- [eval-gate-design.md](./eval-gate-design.md) — the offline gate upstream of shadow.
- [release-artifacts-for-ai.md](./release-artifacts-for-ai.md) — the artifact format including shadow results.
- [prompt-version-pinning.md](./prompt-version-pinning.md) — the pinned prompts the shadow deploys.
- [model-version-pinning.md](./model-version-pinning.md) — the pinned models the shadow deploys.
- [dataset-version-pinning.md](./dataset-version-pinning.md) — the pinned datasets the shadow deploys.
- [model-lifecycle/canary-and-shadow-rollout.md](../model-lifecycle/canary-and-shadow-rollout.md) — model-level shadow mechanic.
- [model-lifecycle/distillation-operations.md](../model-lifecycle/distillation-operations.md) — distillation often validated via shadow.
- [model-lifecycle/ab-model-testing.md](../model-lifecycle/ab-model-testing.md) — A/B as a complement (shadow can be one A/B arm).
- [model-lifecycle/runtime-platform.md](../model-lifecycle/runtime-platform.md) — the runtime the shadow runs on.
- [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md) — the rollback path if shadow-to-canary uncovers issues.
- [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — the judge that scores shadow output.
- [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — eval suite that runs before shadow.
- [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — trace shape capturing shadow vs production.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — the cost signal showing shadow incremental.
- [reliability-engineering/capacity-planning.md](../reliability-engineering/capacity-planning.md) — capacity model including shadow load.

**Cross-repo (architecture sibling):**

- [model-strategy/model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md) — architecture-side migration framing.
- [model-strategy/frontier-vs-open-weights-vs-fine-tune.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/frontier-vs-open-weights-vs-fine-tune.md) — choice context for swaps that warrant shadow.
- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
