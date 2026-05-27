# A/B Model Testing

> **Audience.** Engineers running A/B tests between two models in production traffic. Tech leads who have looked at an A/B dashboard, declared the new model "better," and discovered later that the difference was within noise. Anyone whose A/B testing intuition is from web optimization where the dependent variable is a click and the sample is large. **Scope.** The *engineering* pattern for A/B testing language-model behavior in production: the statistical-significance design for noisy quality metrics, integration with online evals, randomization unit selection, the multi-dimensional "winner" definition that incorporates quality, cost, and latency, and the rollout decision after the test concludes. Pair with [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md) (canary is a *deployment* technique; A/B is a *measurement* technique — different tools with overlapping mechanics). Cross-link to [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) (the live-judge that scores the A/B traffic) and to [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) (the monitoring that detects when an A/B is doing harm). **Worked client.** Meridian Health.

---

## 1. Why this document exists

A/B testing between models is the single highest-leverage decision tool a team has when choosing whether to switch model versions, swap providers, or roll out a fine-tune. It is also the single most-frequently-misused decision tool, because the patterns most teams know come from web experimentation — where the metric is a click, the per-user sample is enormous, the variance is well-understood, and the result reads as a clean significance test. Model A/B testing breaks all four assumptions: the metric is a noisy quality judgment, the per-conversation sample is one, the variance is heavy-tailed, and the result is almost never significant on the first read.

The most common failure modes I see when teams try to A/B model versions are: (1) calling a winner after a day on a quality difference that is well inside noise; (2) randomizing at the wrong unit (per-request when conversation state matters, per-user when the test should target a feature); (3) using a single quality score and declaring a winner without checking that cost or latency moved the other direction; and (4) running the test long enough that the *baseline traffic mix* shifts and the comparison is no longer apples-to-apples. The patterns in this document are about designing the test so the result is real.

A/B testing exists because evals do not perfectly correlate with production performance. Offline evals on the golden set ([eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md)) tell you the model performs differently on the held-out set. They do not tell you the model performs differently on the traffic mix you actually see in production this week. The A/B test is what closes that gap. If the offline eval and the A/B test disagree, the A/B test is closer to truth on the production traffic distribution — and the offline eval is closer to truth on the long-tail distribution the golden set was designed to cover. Both are real; the test is whether you trust the eval enough to skip the A/B, or trust the A/B enough to skip the eval.

This document is opinionated about four things:

1. **A/B testing complements evals; it does not replace them.** Eval the model before the A/B. The A/B answers the production-mix question; the eval answers the held-out-set question. If the eval shows a regression, do not run an A/B to "see how it does in production" — fix the eval first.
2. **Quality is one dimension; cost and latency are coequal.** A model that is 1% better on quality but 30% more expensive or 200ms slower is a different decision from a model that is 1% better and cheaper. The A/B reports all three.
3. **Randomization unit is per-conversation or per-user, not per-request.** Per-request randomization breaks conversation continuity and produces uninterpretable results in any multi-turn feature.
4. **The A/B does not declare a winner; the team does.** The A/B produces evidence. The decision is a business decision, not a statistical one.

Structure: (2) the test design; (3) the randomization unit; (4) the quality metric for an A/B; (5) the statistical significance frame; (6) the cost and latency dimensions; (7) integration with online evals; (8) the run length and stopping rule; (9) the rollout decision; (10) worked Meridian example; (11) anti-patterns; (12) findings; (13) adoption checklist; (14) references.

---

## 2. The test design

An A/B test in model space is structurally similar to web A/B testing but with different assumptions in every cell.

### 2.1 The two arms

- **Control.** The current model, current prompt, current routing rules. This is what production looks like today.
- **Treatment.** The candidate change. Exactly one variable different from control: usually a model version, sometimes a model swap (provider A → provider B), sometimes a fine-tuned variant.

The discipline of "one variable different" is what makes the result interpretable. If the treatment changes the model *and* the prompt, the test cannot distinguish which change caused the difference. That is a useful experiment to run, but it is not an A/B; it is an evaluation of the bundled change.

### 2.2 The hypothesis

State the hypothesis before the test, in writing, with a number attached.

- "We expect the candidate to improve live-judge quality by ≥ 2 points on the 0–10 scale, with no more than 10% increase in p50 latency and no more than 15% increase in cost per request."

A hypothesis with numbers is falsifiable. A hypothesis like "the candidate should be better" is not a hypothesis; it is an aspiration.

### 2.3 The traffic split

- **Production-impacting A/B:** the traffic is split between control and treatment. Both sets of users see real output from one of the two models. Typical split: 50/50 if both are believed safe; 90/10 if treatment is less certain.
- **Shadow A/B:** the treatment runs in parallel against the same traffic but its output is not served to the user. See [canary-and-shadow-rollout.md §4](./canary-and-shadow-rollout.md) for the shadow mechanics. Shadow A/B avoids user-impact risk at the cost of double-billing and the limitation that some metrics (user thumbs-up, conversation continuation) cannot be measured on shadow output.

The choice is pragmatic. If the treatment has been canary-tested and looks safe, run a 50/50 A/B and get the most statistical power per unit time. If the treatment is uncertain, start with shadow until the comparison shows it is safe enough to serve.

### 2.4 What is being measured

Three families of metrics, all measured on the same traffic:

- **Quality.** Live-judge score, golden-set agreement (a subset of A/B traffic that has known-correct answers), user thumbs-up rate, conversation continuation rate, escalation rate, refusal rate.
- **Cost.** Cost per request, cost per conversation, cost per session.
- **Latency.** p50, p95, p99 for first-token and full-response.

Every A/B reports all three. A "winner" is decided on the joint movement of all three.

---

## 3. The randomization unit

The randomization unit is the single most-impactful design decision in the test. Most A/B failures are randomization-unit failures.

### 3.1 Per-request randomization

Each individual request is independently assigned to control or treatment. This is the wrong choice for almost every multi-turn AI feature, because:

- **Conversation continuity breaks.** Turn 1 of a conversation is served by model A; turn 2 is served by model B. The two model responses may disagree, contradict, or differ in tone. The user experience is incoherent.
- **State accumulation across turns differs.** If the system maintains conversational memory or summary state ([context-and-prompt-architecture/chat-history-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/context-and-prompt-architecture/chat-history-architecture.md)), the memory state built up by one model is read by the other. Behavior is the product of A and B, not of A *or* B.
- **The unit of analysis is unclear.** Is one request one data point? Is one conversation one data point? With per-request randomization, both arms see fragments of the same conversation, so neither aggregation works cleanly.

Per-request randomization is appropriate only for stateless, single-turn features (a classifier, a one-shot Q&A, a one-call summarizer).

### 3.2 Per-conversation randomization

A conversation (or session) is assigned at start; all turns within the conversation use the same arm. This is the right choice for most A/B tests of conversational features.

Implementation: on the first turn, the routing layer assigns the conversation to control or treatment, stores the assignment in conversation metadata, and routes every subsequent turn the same way for the lifetime of the conversation.

The conversation is the unit of analysis. The quality metric is aggregated per-conversation, not per-turn.

### 3.3 Per-user randomization

A user is assigned at first encounter; all their conversations use the same arm for the duration of the test. This is appropriate when:

- The metric depends on cross-session behavior (retention, repeat usage, customer satisfaction at the account level).
- Users may compare their experience with peers, and inter-user inconsistency is itself a harm.
- The feature is a B2B product where the unit of harm is a customer account, not a session.

The per-user assignment has lower statistical power per unit of traffic than per-conversation (fewer independent units), but the metric integrity is higher.

### 3.4 Per-tenant randomization

A tenant (a customer organization in a multi-tenant SaaS) is assigned as a unit. This is the right choice for B2B features where:

- Customer success teams interact with the customer per-tenant, and per-user variance would create support chaos.
- The customer expects deterministic behavior; serving the same customer different models depending on which seat is asking is a coherence violation.
- The unit being marketed is "the AI assistant for your team," not "the AI assistant per seat."

Per-tenant randomization has the lowest statistical power per unit of traffic (tenants are few compared to users), but for many B2B systems it is the only defensible unit.

### 3.5 The decision

| Feature type | Randomization unit |
|---|---|
| Stateless classifier or one-shot Q&A | Per-request |
| Conversational support / coordinator | Per-conversation |
| Personal assistant tied to a user account | Per-user |
| B2B AI feature billed and supported per-tenant | Per-tenant |

When in doubt, randomize at the larger unit. The statistical-power cost is real but recoverable (run the test longer); the cost of an uninterpretable result is unrecoverable.

---

## 4. The quality metric for an A/B

The hardest part of model A/B testing is that quality is not a single number, and any single number is a lossy projection of what actually matters.

### 4.1 The live-judge as primary quality metric

Live-judge ([eval-engineering/online-eval-and-feedback.md §3](../eval-engineering/online-eval-and-feedback.md)) is the primary signal. A judge model evaluates each completed conversation (or a sample of them), producing a score on a defined rubric. The judge is the same for both arms (otherwise the comparison is meaningless).

Practical:

- Score every conversation in both arms, or a stratified random sample of ≥ 5000 per arm.
- Use the same judge model and the same prompt for both arms. The judge's prompt is part of the test design, pinned and versioned.
- The judge's own variance matters. Have the judge re-score a held-out sample at the end of the test to verify it has not drifted.

### 4.2 Secondary quality metrics

- **Golden-set agreement.** A subset of A/B traffic (synthetic or seeded queries) has known-correct answers. Both arms are scored against the known-correct, and the rates are compared. This is a noise-free signal at the cost of being on a non-representative subset.
- **User thumbs-up rate.** Where the product collects it. Higher noise floor than live-judge but real user signal.
- **Conversation continuation rate.** Did the user keep talking after the AI's turn? Did they ask a clarifying question (signal of an unclear answer) or close the conversation (signal of an answer that worked or that they gave up)?
- **Escalation rate.** Did the conversation escalate to a human? In Care Coordinator, did the user invoke "talk to a person"?
- **Refusal rate.** Did the model refuse to answer? An over-refusal is a quality regression as much as a wrong answer is.

### 4.3 The composite question

Models rarely beat each other on every metric. A typical result: treatment is +3 points on live-judge, +1% on user-thumbs-up, -2% on conversation-continuation, +5% on escalation. Is treatment the winner?

The decision frame: which metric most reflects the system's goal?

- For Care Coordinator, the goal is *care quality*, proxied by live-judge against a clinical rubric. Conversation-continuation is a weak proxy at best (could mean "the user is satisfied" or "the user is confused and keeps trying").
- For a sales-assist feature, the goal is *conversion*, and the AI's quality is downstream of that.

The composite question is answered by the product owner with input from the engineering data. The engineering team produces the dashboard; the product owner declares the winner.

---

## 5. The statistical significance frame

Model A/B tests are noisier and smaller-sample than web A/B tests. The statistical machinery needs to match.

### 5.1 Sample size

The classical formula for required sample size in a two-arm test:

`n = 16 × σ² / (effect_size)²` *(per arm, for 80% power, 5% significance, two-sided)*

For a live-judge metric with standard deviation 1.5 (on a 0–10 scale) and a hypothesized effect of 0.5 points: `n = 16 × 2.25 / 0.25 = 144` per arm.

For an effect of 0.2 points: `n = 16 × 2.25 / 0.04 = 900` per arm.

Realistic A/Bs target ≥ 1000 conversations per arm.

### 5.2 The noise floor

Live-judge scores are noisy. A single conversation's score can vary by ±2 points depending on the judge's reading. The implication: any effect smaller than the noise floor is invisible regardless of sample size.

The noise floor is measurable. Run the *same* model in both arms (an A/A test) for a small period. The observed "effect size" between the two identical arms is the noise floor. If the A/A test shows 0.3 points of apparent effect, then a real A/B effect of < 0.3 points is invisible.

Always run an A/A *before* the first A/B, and re-run periodically. The noise floor changes when the judge model changes, when the traffic mix changes, or when the rubric changes.

### 5.3 The peeking problem

A/B test results fluctuate during the test. If you check the result every day and stop the test the first day the p-value crosses 0.05, you have inflated the false-positive rate substantially — possibly to 30% or more from the nominal 5%.

The discipline:

- Commit to a sample size before the test starts.
- Do not peek at quality during the test. (Cost and latency dashboards may be checked for safety, but the quality metric is read only at the planned stopping point.)
- If you must peek, use a sequential statistical method (sequential probability ratio test, group-sequential test) that controls for the multiple looks.

### 5.4 The multiple-comparisons problem

If you test five metrics and require one of them to show p < 0.05, your effective false-positive rate is much higher than 5%. The Bonferroni correction (divide α by the number of tests) is conservative but safe; better is to designate one *primary* metric in advance and treat the rest as secondary / exploratory.

---

## 6. The cost and latency dimensions

A quality-only A/B is an incomplete A/B. Cost and latency are coequal.

### 6.1 Cost per request

Both arms record cost per request from the routing layer (provider price × tokens, per [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md)). Aggregate to:

- Mean cost per request per arm.
- Mean cost per conversation per arm.
- 95th percentile cost (large-cost conversations are disproportionate to the bill).

A treatment that improves quality by 1 point but increases cost by 40% is rarely a winner.

### 6.2 Latency dimensions

For chat / streaming features:

- p50 and p95 time to first token.
- p50 and p95 time to last token.
- Total turn latency.

For non-streaming features:

- p50, p95, p99 full-response time.

User-perceived latency is the primary concern; backend latency is secondary.

### 6.3 The trade-off plot

The result dashboard plots both arms on a quality / cost / latency triangle:

- X-axis: quality (live-judge score).
- Y-axis: cost per conversation.
- Bubble size: p95 latency.

If treatment lands up-and-to-the-right (better quality, higher cost), the trade-off is explicit. If treatment lands up-and-to-the-left (better quality, lower cost), it is an unambiguous win. If treatment lands down-and-to-the-right (worse quality, higher cost), it is an unambiguous loss.

### 6.4 The composite winner rule

Define the rule before the test:

- "Treatment wins if Δquality ≥ +0.5 *and* Δcost ≤ +10% *and* Δp95-latency ≤ +50ms."

A composite rule prevents the post-hoc "the quality improved so we shipped it" pattern.

---

## 7. Integration with online evals

The A/B test relies on the live-judge infrastructure described in [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md). The integration:

### 7.1 The judge prompt is pinned

The judge model and judge prompt are pinned for the duration of the A/B. If they change mid-test, the result is invalid (you cannot tell whether the difference is the model or the judge).

### 7.2 Per-arm scoring

The live-judge scores every conversation. The arm label is recorded with the score. The aggregation is per-arm.

### 7.3 Stratified sampling

If live-judge cost is meaningful, sample. Stratify by conversation type so the sample is representative of the traffic mix. A simple random sample is acceptable; a quota sample (e.g., "1000 from each conversation type") is better when types are imbalanced.

### 7.4 Confidence intervals on the score

The live-judge score has confidence intervals from two sources:

- The judge's own variance (the noise floor from §5.2).
- The sample-size noise (standard error of the mean).

The dashboard reports both. A win that is real on point estimate but inside the confidence interval is not yet a win.

---

## 8. The run length and stopping rule

A/B tests run as long as they need to. Most fail because they stop too early.

### 8.1 Minimum run length

- **Statistical minimum.** The sample-size calculation from §5.1.
- **Calendar minimum.** Long enough to cover at least one full weekly cycle. Traffic mix on Monday differs from Saturday; testing only during weekdays produces a biased result for a feature that serves both.
- **Diurnal minimum.** Long enough to cover the day-of-week mix and the hour-of-day mix. For Care Coordinator, peak hours (mid-morning, mid-afternoon) have different traffic patterns than off-peak.

Practical: minimum 7 days. Most A/Bs run 14–28 days.

### 8.2 The stopping rule

Pre-commit:

- **At sample size N, read the result.** If the composite winner rule is met, declare a winner. If not, extend or terminate.
- **At calendar day D (whichever is later), read the result.** Same rule.

Do not stop early because the result "looks good" before the planned read.

### 8.3 Safety stops

Separate from the statistical read, the test has *safety stops* — conditions that abort the test regardless of the planned schedule:

- p95 latency in treatment exceeds SLO. Stop and revert.
- Cost in treatment exceeds budget. Stop and revert.
- Safety / refusal metric in treatment shows severe regression (e.g., 3× refusal rate). Stop and revert.
- User complaints traceable to treatment exceed threshold.

Safety stops are *not* a result; they are an emergency. They are followed by an investigation, not a declaration of winner or loser.

---

## 9. The rollout decision

The A/B produces evidence. The decision is taken by the product owner with engineering and SRE input.

### 9.1 Outcome categories

- **Clear winner.** Composite rule met. Roll treatment to 100% over a planned canary ramp (see [canary-and-shadow-rollout.md §3](./canary-and-shadow-rollout.md)).
- **Clear loser.** Treatment is worse on all dimensions or unambiguously worse on the primary. Discard the treatment, document the result, do not re-test without a real change.
- **Mixed result.** Treatment wins on some metrics, loses on others. The product owner decides whether the trade-off is acceptable.
- **Null result.** No statistically significant difference. The treatment is *not better*. The recommendation is to stay on control unless treatment has other reasons to ship (lower cost is itself a reason).

### 9.2 The promotion path

A winning treatment does not roll out at 100% the moment the A/B ends. It goes through the canary path. The A/B's role was to *justify* the promotion; the canary's role is to *execute* it safely.

### 9.3 Recording the result

Every A/B produces an artifact:

- Test design: hypothesis, randomization unit, sample size, run window.
- Results: per-arm metrics, confidence intervals, primary and secondary outcomes.
- Decision: winner / loser / mixed / null + rationale.
- Owner: who approved.

The artifact is checked into the repository ([cicd-and-eval-gates/release-artifacts-for-ai.md](./)) alongside the release that promoted the winning treatment (or the decision to discard a losing one).

---

## 10. Worked Meridian example: A/B testing the Care Coordinator clinical-knowledge model

Meridian's Care Coordinator has multiple specialized models — supervisor, clinical-knowledge, drafting, classifier — each on a pinned version. The clinical-knowledge model is the primary differentiator of answer quality; it has been on `claude-opus-4-7@2026-04-12` since April. In May, the AI platform team wants to evaluate `claude-opus-4-7@2026-05-15` (a sub-version refresh) as a potential drop-in.

### 10.1 Test design

- **Control:** clinical-knowledge → `claude-opus-4-7@2026-04-12`.
- **Treatment:** clinical-knowledge → `claude-opus-4-7@2026-05-15`. All other models, prompts, and routing rules identical.
- **Hypothesis:** "We expect the new sub-version to improve live-judge quality by ≥ 0.5 points, with cost change within ±5% and p95 latency within ±50ms."
- **Randomization unit:** per-conversation.
- **Split:** 50/50.
- **Sample size:** 5000 conversations per arm (well above the §5.1 minimum given conservative noise floor).
- **Run window:** 14 days, covering two full weekly cycles.

### 10.2 Pre-conditions

Before the A/B:

- Offline eval ran on the golden set; new sub-version shows +1.2 points and no regression on safety / refusal. Acceptable to proceed.
- A/A test ran two weeks earlier on the same traffic mix: observed noise floor of 0.18 points. The hypothesized 0.5-point effect is above the noise floor.
- Live-judge prompt was re-anchored against the rubric two weeks earlier; judge drift confirmed within tolerance.
- Canary at 5% ran for 48 hours on the new sub-version; no safety regressions, no cost spike. Safe to proceed to A/B.

### 10.3 Run

Day 1–14, the A/B runs. Daily monitoring is for *safety only* — p95 latency, cost, refusal rate. Quality is not peeked.

Day 14, read:

| Metric | Control | Treatment | Δ | Inside noise? |
|---|---|---|---|---|
| Live-judge (0–10) | 7.42 | 7.81 | +0.39 | At edge of noise floor |
| Golden-set agreement | 91.3% | 92.6% | +1.3pp | No |
| Thumbs-up rate | 73.1% | 74.0% | +0.9pp | Inside noise |
| Conversation continuation | 41.0% | 40.6% | -0.4pp | Inside noise |
| Escalation rate | 3.4% | 3.2% | -0.2pp | Inside noise |
| Refusal rate | 1.1% | 1.0% | -0.1pp | Inside noise |
| Cost per conversation | $0.082 | $0.083 | +1.2% | n/a |
| p50 latency (first token) | 670ms | 660ms | -10ms | n/a |
| p95 latency (first token) | 1.34s | 1.31s | -30ms | n/a |

### 10.4 Decision

Quality movement on live-judge is right at the noise floor but golden-set agreement (less noisy) is positive at +1.3pp. Cost and latency are flat. The composite winner rule (`Δquality ≥ +0.5 and Δcost ≤ +5% and Δp95-latency ≤ +50ms`) is *not strictly met* on the live-judge but *is met* on the golden-set agreement.

Product owner, with engineering and clinical input: the move from `2026-04-12` to `2026-05-15` is a sub-version within the same major; the offline eval was clean; the A/B shows no regression and a positive (if borderline) signal. Approve promotion with the caveat that the canary should be re-run at 25% for 72 hours before going to 100%.

### 10.5 Findings closed

The A/B retired:

- **ARCH-CARE-052** (the system was on an outdated sub-version with no plan to evaluate the refresh; treated as inertia rather than as a decision).
- **ARCH-CARE-053** (no documented A/B for clinical-knowledge model changes; the May refresh becomes the first one with a recorded artifact).
- **ARCH-CARE-054** (the live-judge had not been anchored against the rubric in the last 90 days; the pre-A/B anchoring closed it).
- **ARCH-CARE-055** (the team had been comparing models on offline eval only; the A/B introduces the production-mix comparison).

---

## 11. Anti-patterns

### 11.1 The "we eyeballed it on a Tuesday" non-test

The team runs the new model on 100% of traffic for a day, looks at the dashboard, and declares it better or worse based on impression. No control arm, no significance, no calendar-cycle coverage. The conclusion is unfalsifiable.

The fix: a real A/B with both arms running concurrently.

### 11.2 The peek-and-stop

The team plans a 14-day A/B, checks the result on day 3, sees treatment ahead, and ships. The 14-day plan was the safeguard; circumventing it inflates false positives.

The fix: pre-commit to the stopping rule. If you cannot resist peeking, use a sequential method that controls for it.

### 11.3 The "no A/A baseline" claim

The team claims a 0.4-point effect is real, never ran an A/A to verify the noise floor. The 0.4-point effect may be entirely noise.

The fix: run an A/A before the first A/B and periodically thereafter.

### 11.4 The quality-only winner declaration

The team finds treatment is +0.5 on quality, declares a winner, ignores that cost is up 30%. Three months later, the finance team finds the increase and asks who approved it.

The fix: composite winner rule pre-committed, all three dimensions in the dashboard.

### 11.5 The bundled-change A/B

The team changes the model *and* the prompt and *and* the retrieval corpus and runs one A/B. The treatment wins, but they cannot tell which change was responsible. They cannot re-derive the win when the bundle changes.

The fix: one variable per A/B. Test changes sequentially. If you must bundle for time pressure, document that the result is for the bundle, not for any single component.

### 11.6 The randomization-unit mistake

The team randomizes per-request for a conversational feature. Conversation continuity is broken; users see incoherent responses; quality scores reflect the incoherence, not the model.

The fix: randomization unit per-conversation or larger for any feature with state.

### 11.7 The judge-prompt change mid-test

The team improves the live-judge prompt midway through the A/B. The score distribution shifts; the comparison is invalid.

The fix: pin the judge for the duration of the test. Improvements to the judge wait for the next A/B.

### 11.8 The "we'll fix it post-launch" rollout

The A/B is null or slightly negative; the team ships anyway because "we're committed to the new model." The A/B was decoration.

The fix: pre-commit the decision rule. If treatment does not meet the rule, do not ship.

---

## 12. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| ML-AB-001 | A/B tests run without an A/A baseline; noise floor unknown | High | Run quarterly A/A on production traffic; publish noise floor per metric | AI Platform |
| ML-AB-002 | Randomization unit varies between tests with no documented rationale | High | Document randomization-unit decision in the test design; default to per-conversation | AI Platform |
| ML-AB-003 | A/B results read only on quality; cost and latency not in dashboard | High | Add cost and latency arms to A/B dashboard; require composite rule | AI Platform + FinOps |
| ML-AB-004 | Stopping rule not pre-committed; team peeks daily | High | Pre-commit sample size and stopping rule in test design doc; ban mid-test peek for quality | AI Platform |
| ML-AB-005 | Live-judge prompt changes during running A/Bs | High | Freeze judge config for the test window; document at start | Eval Eng |
| ML-AB-006 | Sample size not derived from effect-size calculation | High | Compute n from §5.1; record in test design before launching | AI Platform |
| ML-AB-007 | A/B runs only on weekdays / business hours; weekend traffic excluded | Medium | Minimum 7 days including a full weekly cycle | AI Platform |
| ML-AB-008 | Multiple metrics compared without correction; p-hacking risk | Medium | Designate one primary metric per test; correct or downgrade the rest to exploratory | AI Platform |
| ML-AB-009 | Bundled changes tested as one A/B; cannot decompose result | Medium | One variable per A/B; sequence the changes | AI Platform |
| ML-AB-010 | A/B results stored in chat / docs only; no canonical record | Medium | Store A/B artifact in repo per [release-artifacts-for-ai.md](../cicd-and-eval-gates/) | AI Platform |
| ML-AB-011 | Safety-stop thresholds not pre-defined | High | Pre-define p95-latency, cost, refusal-rate stop thresholds; integrate with monitoring | AI Platform + SRE |
| ML-AB-012 | Treatment shipped on borderline-significant result without product approval | Medium | Decision is the product owner's; engineering produces evidence | Product + AI Platform |
| ML-AB-013 | No documented decision rule for mixed-result outcomes | Medium | Define rule pre-test; the rule is part of the design artifact | Product + AI Platform |
| ML-AB-014 | A/B traffic split asymmetric without justification | Low | 50/50 default; deviate only with documented reason (typically treatment uncertainty) | AI Platform |
| ML-AB-015 | A/B-eligible features not flagged; tests run in production with full traffic | Medium | Maintain a registry of A/B-eligible features; integrate with routing | AI Platform |
| ML-AB-016 | Stratified sampling not used; aggregate result hides per-segment regression | Medium | Stratify by conversation type and per-tenant tier; report per-segment | AI Platform |
| ML-AB-017 | Calendar-time minimums not enforced; tests stop after sample-size hit even on day 2 | High | Enforce both n ≥ target *and* days ≥ 7; whichever is later | AI Platform |
| ML-AB-018 | A/A test not re-run after judge or rubric changes | Medium | Re-baseline noise floor when judge model, prompt, or rubric changes | Eval Eng |

---

## 13. Adoption checklist

- [ ] A/A baseline run on production traffic mix; noise floor per metric published.
- [ ] Standard test-design template in use: hypothesis, randomization unit, sample size, run window, composite winner rule, safety stops.
- [ ] Randomization-unit decision documented per feature type (per-request / per-conversation / per-user / per-tenant).
- [ ] Sample size derived from effect size; no test launches without the calculation.
- [ ] A/B dashboard shows quality, cost, and latency in a single view with confidence intervals.
- [ ] Live-judge config (model, prompt, rubric) pinned for the test duration.
- [ ] Pre-committed stopping rule; no quality peeking during the test.
- [ ] Safety stops defined and wired to monitoring.
- [ ] Primary metric designated; secondary metrics treated as exploratory.
- [ ] Calendar minimum enforced (≥ 7 days covering a full weekly cycle).
- [ ] One variable per A/B; bundled changes documented as bundle-level evaluations.
- [ ] A/B artifact (design + result + decision) stored in repo alongside the promoting release.
- [ ] Mixed-result decision authority is the product owner with engineering / SRE input.
- [ ] Winning treatment promoted through canary, not via direct cutover.
- [ ] Quarterly A/A re-runs; noise floor refreshed when judge or rubric changes.

---

## 14. References

**Internal:**

- [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md) — the deployment mechanic that executes a winning A/B; shadow mode is one A/B technique.
- [rollback-procedures.md](./rollback-procedures.md) — the safety net for a winning A/B that misbehaves at full traffic.
- [model-promotion.md](./model-promotion.md) — the promotion pipeline that a winning A/B feeds.
- [model-registry.md](./model-registry.md) — the pinned versions both arms reference.
- [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — the live-judge that scores A/B traffic.
- [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — the offline eval that runs before any A/B.
- [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — the pre-A/B sanity check.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — the monitoring that may force a safety stop.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — the cost-per-request signal for the A/B.
- [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md) — when a safety stop becomes an incident.

**Cross-repo (architecture sibling):**

- [model-strategy/model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md) — the architecture context for *why* an A/B is being run.
- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
