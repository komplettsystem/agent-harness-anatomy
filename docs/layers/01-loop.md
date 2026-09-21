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

**Cutting round-trips, not just what's in them.** A different kind of efficiency question than the ones later layers cover: not what enters the context window, but how many model calls a task takes to finish. NVIDIA Labs' SoL-Pi (MIT-licensed, installs on top of the Pi coding harness via its extension interfaces, every mechanism opt-in) packages four such mechanisms, discovered through NVIDIA's own scaled auto-research loops. The one that belongs here is Action Fusion: combine a file edit with its follow-up command into one model call instead of two. NVIDIA reports 45–49% fewer tokens across all four mechanisms combined, at roughly 94% of task performance retained on their own benchmarks — not independently reproduced here. The other three mechanisms belong to [Layer 3](03-context.md) and [Layer 5](05-verification.md).

**Checkpointing inside a run.** Typed state written at step boundaries, so a run resumes after an interruption and can be replayed for debugging. This is the single-run version of what [Layer 7](07-control-plane.md) does with durable execution across runs. Without it, every crash costs the whole run — which matters little for a 20-second task and a great deal for a 40-minute one.

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

**None worth billing for.** Confirm the loop is model-agnostic, or that switching is cheap, and move on. Every hour spent comparing loops is an hour not spent on Layers 2, 4, 5 and 8, which is where your project will actually succeed or fail.
