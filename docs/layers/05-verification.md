# Layer 5 — Verification

## The problem

"Agents that improve with feedback" is the central claim of the category. Whether it is true for *your* process depends entirely on one thing: whether the output can be checked, how cheaply, and how reliably. Verification is whatever judges the output, plus the machinery that turns that judgment into improvement — retry, memory update, prompt or skill revision.

This is the layer that decides everything above it. Coding is ahead of most other domains not because code is easy but because it is cheap to check. An agent that writes code can run it seconds later and get a clear answer from software, with no person involved: the automated tests pass or fail, the program starts or crashes with an error, and a style checker either accepts the code or lists what is wrong. That answer is free, immediate, and the same no matter who asks. Most business work gets none of this by default. Nothing tells you automatically whether a contract review missed a clause, whether a campaign plan will work, or whether a customer letter strikes the right tone. Someone has to read it and judge, or you wait weeks for the results to show up.

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

**Why the separation works.** Prithvi Rajasekaran at Anthropic observed that, asked to judge their own output, agents "tend to respond by confidently praising the work—even when, to a human observer, the quality is obviously mediocre." He reports that "tuning a standalone evaluator to be skeptical turns out to be far more tractable than making a generator critical of its own work" (["Harness design for long-running application development"](https://www.anthropic.com/engineering/harness-design-long-running-apps), March 2026).

**Grading work that no software can check.** Rajasekaran's harness also shows how to build a rubric grader for subjective work. Name the criteria that turn "is this good?" into something gradable; for frontend design he used design quality, originality, craft and functionality. Weight them toward where the model is weak: Claude "already scored well on craft and functionality by default", so design quality and originality counted for more. Then calibrate the grader with a few examples scored in detail, so its scores match your own judgement and stay consistent from one round to the next. For many business processes, a written rubric and a handful of scored examples is a realistic first verifier, long before any automated check exists.

**Let the grader use the result, not read about it.** The evaluator in that harness clicked through the running application with a browser tool, the way a user would, and checked the database state behind the interface. A grader that reads only the agent's own record of what it did is judging the agent's account of the work, and one that reads only the code cannot see whether it runs. Using the app checks both.

**Agree on "done" before the work starts.** Before each chunk of work, the generator and evaluator "negotiated a sprint contract: agreeing on what 'done' looked like for that chunk of work before any code was written." Its purpose, in Rajasekaran's words, was "to bridge the gap between user stories and testable implementation". User stories are short descriptions of what a user needs; the spec stayed at that level, and each contract turned a piece of it into something the evaluator could test.

**An evaluator costs money, and its value moves with the model.** One test prompt asked for a game-making app. A solo run took 20 minutes and $9, and in its game nothing responded to input. The full planner, generator and evaluator harness took 6 hours and $200; its game had rough edges, but the author could move his character and play it. When Opus 4.6 arrived, Rajasekaran removed the sprint structure but kept the planner and evaluator, concluding that "the evaluator is worth the cost when the task sits beyond what the current model does reliably solo." Budget an evaluator for work at the edge of what the model can do, and check where that edge is each time the model changes. All of this comes from one author's experiments in two domains, so the $9 and $200 runs are one data point about overhead.

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

**Standalone, runtime-neutral.** Verification is not only sold inside agent platforms. Braintrust is a hosted evaluation and observability product that works across model providers: datasets and scorers (exact match, custom code, LLM-as-judge) run separately from the task, and its trace and eval features cover whole agent runs, tool calls included. The data plane can be self-hosted while the control plane stays hosted. I checked Braintrust against its own site and docs on 2026-09-21; its pricing tiers are not verified. Galileo, Arize Phoenix, MLflow and DeepEval make similar runtime-neutral claims, and I have not checked them. So the platforms that lead on this layer bundle it, but the bundle is not the only route, and a neutral product is one way to keep agent definitions portable.

## The business-case question

**How cheaply and reliably can the output be checked?** The answer sets the autonomy ceiling, determines whether memory helps or poisons, and decides whether this process is a candidate for an agent at all. Everything else on the [diagnostic](../business-case-diagnostic.md) is downstream of it.
