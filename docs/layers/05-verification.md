# Layer 5 — Verification

## The problem

"Agents that improve with feedback" is the central claim of the category. Whether it is true for *your* process depends entirely on one thing: whether the output can be checked, how cheaply, and how reliably. Verification is whatever judges the output, plus the machinery that turns that judgment into improvement — retry, memory update, prompt or skill revision.

This is the layer that decides everything above it. Coding is ahead of every other domain not because code is easy but because its verifiers are free, fast and objective: tests pass or fail, the compiler accepts or rejects, the linter is not negotiable. Most business processes have none of those properties by default. Contract review, campaign planning, customer correspondence, research synthesis — none of them come with a compiler.

If you take a single question from this entire map into a scoping conversation, take this one: *how cheaply and reliably can the output be checked?* If the honest answer is "we can't," then the first deliverable is a verifier, not an agent. That reordering is usually unwelcome and almost always correct.

## The verifier ladder

From cheapest signal to most expensive:

| Verifier | Speed | Objectivity | Example |
|---|---|---|---|
| Deterministic check | Instant | High | Schema valid, totals reconcile, a confirmation number comes back |
| Rubric grader (LLM, separate context) | Seconds | Medium | Draft scored against editorial or compliance guidelines |
| Human approval at the action boundary | Minutes to hours | High, but costly | Approve before sending, paying, or filing |
| Fleet pattern review (out-of-band) | Batch, e.g. nightly | Medium — finds recurring failures, not correctness | A scheduled job notices the same tool misconfiguration failing across many sessions |
| Sampled human review | Days | High | QA on 5% of outputs |
| Business outcome | Weeks to months | Highest, but noisy and lagging | Conversion, churn, claim acceptance |

Most real systems use several rungs at once: a deterministic check on every run, a rubric grader on every output, human approval at the irreversible boundary, and sampled review for calibration. The design question is which rung carries the weight, not which one you pick.

## Concepts worth owning

**Separate the grader from the worker.** The grader runs in its own context, so the agent's own reasoning cannot bias the judgement. This sounds obvious and is violated constantly — including in otherwise well-designed self-improvement loops where the same agent that ran the iteration also scores it. Claude Managed Agents productises the separation as "outcomes": you define success as a rubric, a separate grader in its own context scores the work, and the agent iterates until it passes.

**Evals versus guardrails.** Evals *measure* quality; guardrails *block* actions. Different layers, different owners, different review cadence. Guardrails live in [Layer 8](08-governance.md) because blocking is a policy decision. Conflating the two produces systems that measure a great deal and prevent nothing.

**Trace-linked evals.** A score is nearly useless unless you can jump from the score to the exact production trace that produced it. Microsoft built Foundry's control plane around this property: tracing runs on OpenTelemetry and each eval result links back to its production trace. It is a good bar to hold any evaluation stack against, including a homegrown one.

**Fleet-level verification, out of band.** The most interesting new pattern of 2026. A batch process reviews many transcripts (including tool calls and metadata) against the current memory store. An orchestrator fans the review out to subagents, keeps only the patterns prevalent enough to matter, and proposes memory changes backed by example transcripts and prevalence statistics. A human accepts or rejects each change. Anthropic's analogy for its "dreaming" implementation is a head teacher who reviews all the students' work and fixes the *curriculum* rather than each answer.

Two things can go wrong with it. It detects *recurring* failure, which is not the same as incorrectness — it still needs an external definition of "good," and it will happily converge on a consistently wrong practice. And commentators have warned that it can turn bad habits into permanent policy. Treat accepted changes like code review, not like autosave.

**Optimization loops can game the grader.** If an agent tunes itself against a rubric, it will eventually optimise the rubric rather than the work. The community pattern around Hermes Agent's self-evolution is instructive: a nightly self-improvement run using DSPy + GEPA to optimise skills and prompts against a metric, plus a *separate verification job* whose only purpose is to stop the loop gaming its own optimisation. If you build a self-improving agent without that second job, you have built a metric-gaming machine with extra steps.

**Delegate the reading, keep the judging.** NVIDIA Labs' SoL-Pi (see [Layer 1](01-loop.md)) has a fourth mechanism worth knowing here: Evidence-Preserving Reducer routes log-reading to a cheaper model, but the evidence the primary agent needs to verify still reaches it — the delegation cuts cost without cutting the verification step. Same discipline as "separate the grader from the worker," above, applied to a different seam: who reads the evidence, not just who judges it.

## The landscape

**Open source.**

| Project | What to look at |
|---|---|
| DSPy + GEPA (Stanford NLP) | Optimising prompts and programs against a metric; used in Hermes Agent self-evolution pipelines |
| Inspect (UK AI Safety Institute) | Eval harness design done properly: tasks, solvers, scorers as first-class separable pieces |
| Langfuse | Tracing plus evals on production traffic, which is where the interesting failures are |

**Managed.** Two platforms are graded full coverage on this layer. **Claude Managed Agents:** rubric plus independent grader ("outcomes"), and dreaming as a fleet-level reviewer, with the caveat that dreaming is a research preview. **Microsoft Foundry:** the Agent Optimizer does rubric-based evaluation and suggests fixes, with trace-linked results; confirm which evaluation features are GA rather than preview before relying on them commercially.

**AWS AgentCore is graded partial here, and that is its notable gap** relative to how complete the rest of its stack is: check what evaluation tooling ships natively before assuming it is covered. **Google Vertex Agent Engine is also partial**, and its native evaluation offering is one of the three things I would verify first about that platform. **The OpenAI Agents SDK provides nothing on this layer** — it is a library, and verification is left entirely to you.

One open question I have not found a good answer to: does anyone offer verification as a standalone, runtime-neutral product, or is it being absorbed into every platform as a lock-in mechanism? The current evidence points to absorption, which is a strategic problem for anyone who wants to keep their agent definitions portable.

## The business-case question

**How cheaply and reliably can the output be checked?** The answer sets the autonomy ceiling, determines whether memory helps or poisons, and decides whether this process is a candidate for an agent at all. Everything else on the [diagnostic](../business-case-diagnostic.md) is downstream of it.
