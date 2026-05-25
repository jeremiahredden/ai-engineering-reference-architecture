# Prompt Engineering

## What this folder is

The engineering discipline of treating prompts as code: versioned, reviewed, tested, deployed, and rolled back like the rest of the codebase. The material here is what I put in front of a team when the question is: *the prompt has 1,200 tokens, lives as a string in a notebook, has been edited 40 times in the last quarter, and nobody can tell us which version is in production — how do we get from here to engineered prompts?*

## The organizing principle

Prompts are software. They are the largest single behavioral lever a team has over an AI system; they fail in production; they have regressions; they have backwards-compatibility concerns; they need ownership. But most teams treat prompts as configuration at best and as notebook strings at worst — committed inline with no review, no versioning, no test suite, no rollback path. The result is the prompt that nobody dares change because nobody knows what depends on it.

So the patterns here treat the prompt as a *versioned artifact with an owner and a test suite*. The mechanics — pulling prompts out of code into a prompt store, versioning them, pinning versions in releases, running evals against changes — are not exotic. They are the same engineering disciplines applied to a different artifact. The work is getting a team to do them consistently.

The folder is opinionated about three things specifically. First, that prompts belong outside the code they are invoked from (in a prompts directory, a prompt store, or a versioned config), not as inline strings. Second, that any change to a prompt that ships to production must pass the same eval gate as the code that ships to production. Third, that prompts have an *API contract* with their consumers (the chains, evals, parsers, and downstream extractions that depend on the prompt's output shape), and breaking that contract is a breaking change with the same lifecycle as breaking an HTTP API.

## Planned documents

- **prompts-as-code-discipline.md** *(coming)* — Pulling prompts out of inline strings into a versioned store, the directory / module structure, the ownership pattern (who reviews prompt PRs), the change-control workflow, and the migration path from "prompts inline" to "prompts as artifacts" without freezing the team.
- **prompt-versioning.md** *(coming)* — Semantic versioning for prompts, pinning prompt versions in releases, the prompt-changelog pattern, backwards-compatible prompt evolution, the deprecation lifecycle for retired prompts, and the integration with the engineering sibling's CI / CD.
- **structured-output-engineering.md** *(coming)* — JSON Schema with tool calls, constrained decoding (Outlines, Instructor, Guidance), output validation with repair-loops, the failure modes of structured output (truncation, schema drift, hallucinated fields, escape-character issues), and the engineering patterns that make structured output reliable enough to depend on.
- **few-shot-engineering.md** *(coming)* — When few-shot examples earn their tokens, example selection (curated vs retrieved), example ordering, the diminishing returns curve, and the migration from few-shot to fine-tune when few-shot example budget grows past about 20 examples or the workload has settled.
- **prompt-libraries-and-components.md** *(coming)* — The componentization of prompts: shared system-prompt fragments, reusable instruction blocks, formatter modules, the platform-level prompt library pattern, and the per-feature overlay model that lets multiple features share a base without each one needing to re-derive it.
- **prompt-ab-testing.md** *(coming)* — Running prompt experiments in production: traffic split, statistical-significance design for noisy quality metrics, integration with online evals (sibling `eval-engineering/`), interaction with rollback, and the discipline that prevents A/B tests from accumulating into a permanent forking jungle.
- **prompt-as-api-contract.md** *(coming)* — Treating the prompt's output as a versioned API for everything downstream: parsers, evals, structured extractors, chained prompts. Breaking change classification, deprecation lifecycle, the consumer-driven contract test pattern adapted to prompts.
- **prompt-anti-patterns.md** *(coming)* — The six prompt anti-patterns I see most often: inline-string-with-40-edit-history, "let's just add a sentence" without eval, single-monolith-prompt-for-everything, copy-paste-and-diverge across features, no-version-pinning-in-releases, and prompt-as-config-without-treating-config-changes-as-code-changes.

## How to use this section

**If your prompts live as inline strings in code**, `prompts-as-code-discipline.md` is the first move. The refactor is small and the leverage is large.

**If you have prompts in a store but no versioning**, `prompt-versioning.md` is the next move. Versioning is what makes rollback possible.

**If your prompt changes occasionally break downstream consumers** (a parser that depends on a specific output format), `prompt-as-api-contract.md` is the discipline you need. The fix is contract-test patterns, not "be more careful."

## What this section is not

- **A "write better prompts" tutorial.** How to write good system prompts, chain-of-thought, role priming, few-shot example design — every model vendor publishes those (Anthropic's prompt engineering guide, OpenAI's, Google's), and they are vendor-specific. This folder is about the *engineering practice* around prompts, not about prompt-writing technique.
- **A prompt-injection defense guide.** Adversarial prompt injection lives in the sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture)'s `llm-application-security/` folder.
