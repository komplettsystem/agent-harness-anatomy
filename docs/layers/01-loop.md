# Layer 1 — Loop

## The problem

A language model produces text. A business process needs actions taken against real systems, in sequence, with each action informed by the result of the last one. Something has to sit between the two: assemble a prompt, call the model, parse out the tool calls, execute them, append the results, and go round again until the work is done, the user stops it, or something breaks. That something is the loop, and it is the only layer that is strictly mandatory. Everything else in this map is an answer to a question the loop raises.

The good news for anyone scoping an agent project: this layer is a commodity. It was the hard part in 2024 and it is a solved, well-documented problem now, available in half a dozen good open-source implementations and inside every managed platform. If a vendor's differentiation story is mostly about their loop, they are selling you 2024.

## Concepts worth owning

**Agent versus workflow.** A workflow has a fixed path; an agent chooses its path at runtime. This is the single most consequential design decision on the layer, and most teams get it wrong in the direction of too much agency. Most business processes want a workflow with agentic steps inside it (deterministic control flow, with a model doing the judgement-heavy bits), not a free agent picking its own route through your ERP. Free agency is appropriate where the search space genuinely cannot be enumerated in advance; it is expensive and unpredictable everywhere else.

**Modes.** A read-only planning agent and a full-access build agent, with a human switch between them, is the simplest safety pattern that exists. It costs almost nothing to implement and it converts an entire class of "the agent did something irreversible" incidents into "the agent proposed something wrong and I said no." OpenCode ships this as `plan` and `build`. If a harness does not offer a read-only mode, that is a real gap, not a stylistic difference.

**Client/server split.** The loop runs as a server with an API; terminals, chat apps, IDEs, web UIs and scripts are all just clients of it. This is the architectural move that makes a coding harness reusable outside coding, and it is worth checking for explicitly: a harness that is welded to its terminal UI cannot be driven from a webhook or a ticket-state change, which is how most business agents are actually triggered.

**Model-native harnesses.** Vendors increasingly tune the loop to their own models' habits — prompt scaffolding, tool-call formats, stop conditions. This buys reliability and costs portability. It is a legitimate trade, but make it knowingly, and check what switching would cost before you are committed.

**Failure taxonomy.** Four kinds of error, each with a different owner:

| Kind | Example | Correct handling |
|---|---|---|
| Transient | Rate limit, network blip | Harness retries, silently |
| Model-recoverable | Wrong argument, file not found | Hand the error back as an observation; let the agent adapt |
| User-fixable | Missing credential, no permission | Stop and ask |
| Unexpected | Harness bug, corrupt state | Halt and log; do not retry |

Misclassification here is what makes an agent burn an entire context window looping on a permission error it can never resolve. This is the most common preventable failure mode in production agent systems and the easiest one to test for.

**Cutting round-trips, not just what's in them.** A different kind of efficiency question than the ones later layers cover: not what enters the context window, but how many model calls a task takes to finish. NVIDIA Labs' SoL-Pi (MIT-licensed, installs on top of the Pi coding harness via its extension interfaces, every mechanism opt-in) packages four such mechanisms, discovered through NVIDIA's own scaled auto-research loops. The one that belongs here is Action Fusion: combine a file edit with its follow-up command into one model call instead of two. NVIDIA reports 45–49% fewer tokens than Pi with all four mechanisms combined, and roughly 94% of Pi's average score on EdgeBench (its own 51-task suite, two model backends). On its second benchmark, Terminal-Bench 4, SoL-Pi solved 15 of 63 tasks against 18 for both Pi and Codex. These are NVIDIA's own results and I have not reproduced them. The other three mechanisms belong to [Layer 3](03-context.md) and [Layer 5](05-verification.md).

**Checkpointing inside a run.** Typed state written at step boundaries, so a run resumes after an interruption and can be replayed for debugging. This is the single-run version of what [Layer 7](07-control-plane.md) does with durable execution across runs. Without it, every crash costs the whole run — which matters little for a 20-second task and a great deal for a 40-minute one.

**Stopping is not succeeding.** A run can end because the task is verifiably done, a policy blocked it, a budget ran out, a human is needed, or someone cancelled it. The harness should record these as different end states. A loop that ended on its turn limit has proven nothing about the task, and if every ending is logged as one generic "done", [Layer 5](05-verification.md) cannot tell a timeout from a success. Each budget (turns, time, cost, tool calls) is a decision someone made about what the task is worth.

**Routing tasks to models.** The loop decides which model each call goes to. Models from different providers are good at different things and differ widely in price, so that choice is a design decision, not a default. It can be made in three places. A gateway in front of the providers picks by rules such as cost, latency, or fallback when a provider is down; LiteLLM (self-hosted, over 100 providers behind one API) and OpenRouter (hosted) are the common ones. A learned router predicts the cheapest model that is good enough for each request; the best-known open one, RouteLLM, reported costs cut "by over 2 times in certain cases" on standard benchmarks, but its repository has had no commit since August 2024. Or the loop delegates by role: a strong model plans and a cheaper one executes (Claude Code's `opusplan` setting uses Opus in plan mode and Sonnet for execution), subagents each get their own model, or a main model consults a stronger advisor model. A June 2026 preprint tested on coding tasks, Agent-as-a-Router (arXiv 2606.22902), finds that static routers lack information about how models actually perform and break down on unfamiliar tasks. It makes routing a loop of its own, with a verifier and a memory of which model did well on which kind of task. The results are the authors' own.

What routing is worth depends on how the deployment pays for models. With per-token billing it can lower the bill, but only cost per finished task counts, not price per token: a model that thinks longer or needs more turns can cost more per task than its price suggests, in either direction, so measure it on the client's own tasks. On a flat subscription the invoice stays the same, but routing can still matter for how far the plan's usage limits go, and for putting each task on the model that handles it best. Before either, settle which providers may see the client's data at all ([Layer 8](08-governance.md)); every provider added to the routing table is one more.

## The landscape

**Open source.**

| Project | License / language | What to look at |
|---|---|---|
| OpenCode | MIT, model-agnostic; terminal, desktop, IDE, plus a local server with an HTTP API | The most readable reference implementation of a production loop. Built-in `build` / `plan` agents, a `@general` subagent for multi-step search in its own context, project rules in a committed `AGENTS.md`. Widely adopted. |
| OpenAI Agents SDK | MIT, Python first with TypeScript following | The cleanest articulation anywhere of the split between the harness (control: instructions, tools, approvals, tracing, handoffs, resume bookkeeping) and the sandbox (execution). Since its April 2026 update it ships a "model-native" harness — the loop OpenAI uses in Codex, packaged. |
| Strands Agents | Apache-2.0, Python and TypeScript, AWS | The engine underneath AgentCore's managed harness. Worth reading if you want to understand what the managed version is actually doing. |
| LangChain Deep Agents | MIT, Python | A "batteries-included" harness. Its public reference build is useful less for the loop than for how it stacks context — see [Layer 3](03-context.md). |

**Managed.** AWS AgentCore's managed harness (declare model, tools and instructions; AWS runs the loop; in preview as of 2026-09-11), Claude Managed Agents (a Claude Code-style harness run as a service), and Microsoft Foundry hosted agents (package as a container, declare in a manifest, deploy) all own this layer completely. All three are graded full coverage on Layer 1 in the [comparison](../framework-comparison.md).

## The business-case question

**None worth billing for, with one exception.** Decide whether tasks get routed to different models; the answer follows from how the client pays for and is limited on model use, and which providers are allowed to see their data. Beyond that, confirm the loop is model-agnostic, or that switching is cheap, and move on. Every hour spent comparing loops is an hour not spent on Layers 2, 4, 5 and 8, which is where your project will actually succeed or fail.
