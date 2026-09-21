# Layer 7 — Control plane

## The problem

Everything above a single run. Which agent handles which request; through which channel it arrives; on what schedule it fires; how it coordinates with other agents; and whether the work survives a crash. A demo is one agent, one task, one chat window. A deployed system is a standing capability that receives work from Slack, a webhook, a cron entry and a ticket state change, runs it across restarts, and hands off between specialists.

This is a genuinely different purchase from the loop, and it is where "we built a great prototype" most often fails to become "we run this."

## Concepts worth owning

**Gateway-first versus agent-first.** Two defensible architectures. OpenClaw wraps agents in a messaging and routing gateway — channel adapters for Slack, WhatsApp, Discord and web chat, plus identity, session lifecycle and authorization, with the agent runtime behind it. Hermes Agent does the reverse: a gateway wrapped around one deep, learning agent. Orchestration breadth versus single-agent depth. Which one you want follows from whether your problem is "many surfaces, many kinds of request" or "one process, done increasingly well."

**Sessions as shared, first-class objects.** A design point worth calling out because it is often conflated with multi-agent orchestration and is not the same thing: a session that multiple humans can join, leave, or take over while it keeps its context. OpenClaw 2.0 (v2026.8.1, released 30 August 2026) shipped this as shared cloud sessions. The architectural consequence is that a session is addressable independently of whichever human is currently attached to it — which is what separates a collaborative workspace from a private assistant chat. It also raises an unresolved question that nobody has a clean answer to: whose credentials does an action taken in a shared thread carry?

**Lead agent plus subagents.** A cheap, fast lead agent triages and delegates to stronger, more expensive subagents running in parallel with isolated context. The cost profile is good and the context hygiene is better (see [Layer 3](03-context.md)). Claude Managed Agents productises this shape directly; LangChain's published reference build uses one subagent per external platform.

**Durable execution.** Long-running work must survive restarts: checkpoints, event sourcing, replay. Temporal and LangGraph checkpointers are the mature open-source answers. Keeping durable state in a control-plane database rather than in the workspace is the pattern to copy — a workspace that is stopped or rebuilt then resumes from last known state rather than from nothing.

**Triggers.** Chat message, cron, webhook, ticket state change, inbound email. **Business agents are mostly event-driven, not chat-driven**, which is worth saying loudly because almost every vendor demo is chat-driven. A harness that cannot be invoked by a webhook is not a candidate for most real deployments, whatever it looks like in the video.

**When not to go multi-agent.** It costs tokens and adds failure modes — handoff loss, duplicated work, cascading misunderstandings. Use it for genuine parallelism or for context isolation. Do not use it to reproduce your org chart in software; the org chart is a coordination artefact for humans with limited attention, and it does not transfer. None of this is an argument against subagents — it's an argument for making each dispatch cost-effective without lowering what it produces. NVIDIA Labs' SoL-Pi (see [Layer 1](01-loop.md)) is a concrete worked example: cut round-trips and re-emitted context, but never skip the evidence a verification step needs.

## The landscape

**Open source.**

| Project | License / runtime | What to look at |
|---|---|---|
| OpenClaw 2.0 | MIT, TypeScript/Swift, self-hostable | The strongest available template for the Gateway → Agent Runtime → Skills → Storage shape. Gateway routes to isolated agents, each with its own workspace, tools and memory. Storage moved from flat files to SQLite in 2.0. A small trusted kernel with everything else as plugins. Microsoft built its enterprise agent Scout on it (announced at Build 2026), which is a notable signal — an open gateway and a managed platform coexisting inside one vendor's strategy. |
| LangGraph | Python, JS | Explicit state graphs, checkpointing, human interrupts. The most legible model of "workflow with agentic steps." |
| Temporal | Server | Durable execution for workflows containing agentic steps. The boring, correct answer for anything that must not lose work. |
| A2A | Protocol (originated at Google) | The agent-to-agent seam, for when coordination crosses an organisational or vendor boundary. |

**The same pattern, sold no-code.** Taskade's "AI Teams" (multiple agents collaborating on a shared task via handoffs and peer review) is the gateway-first idea repackaged for business users rather than engineers, alongside event-triggered automations across a large integration catalogue. I track it as evidence of how far down the org this pattern is spreading, not as a design to imitate.

**Managed.** Microsoft Foundry is the only platform graded full coverage on this layer, via routines, long-running agents, and publication into Teams and Microsoft 365 Copilot. Claude Managed Agents (lead-plus-subagent orchestration and webhooks), AWS AgentCore (code-defined multi-agent on Runtime) and Google Vertex Agent Engine are all graded partial. The OpenAI Agents SDK provides nothing here — handoffs exist in the library, but hosting, scheduling, channels and durability are yours.

## The business-case question

**Is this one agent doing one task on request, or a standing team working across channels and schedules?** The second is a different purchase, a different operational commitment, and usually a different budget line. Deciding it late is expensive, because the answer determines whether you needed a library or a platform.
