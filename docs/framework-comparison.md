# Framework and platform comparison

Eleven products and protocols, placed against the [eight layers](../README.md#the-eight-layers). For each one: which layers it covers, who I think it is actually built for, and where its real gap is.

**How to read this, and what it is not.** Two tables follow, and the difference between them is deliberate. The first covers the five general-purpose agent platforms I assessed with per-layer depth grades. The second covers six products and protocols that address a defined subset of layers but for which I did not produce depth grades — listing the layers they *address* is an honest statement of scope, and claiming graded coverage for them would not be. Do not read a blank in the second table as a zero in the first.

All facts were verified against primary sources on the dates stated. Vendor capabilities in this category change monthly. Re-verify before quoting any of it.

---

## Table 1 — Graded platforms

● full · ◐ partial or preview · ○ not provided

| Layer | Claude Managed Agents | AWS AgentCore | Google Vertex Agent Engine | Microsoft Foundry | OpenAI Agents SDK |
|---|---|---|---|---|---|
| 1 Loop | ● | ● | ◐ | ● | ● |
| 2 Tools & integrations | ◐ | ● | ◐ | ● | ◐ |
| 3 Context | ● | ◐ | ◐ | ◐ | ◐ |
| 4 Memory | ● | ● | ● | ● | ◐ |
| 5 Verification | ● | ◐ | ◐ | ● | ○ |
| 6 Execution environment | ● | ● | ◐ | ● | ◐ |
| 7 Control plane | ◐ | ◐ | ◐ | ● | ○ |
| 8 Governance | ◐ | ● | ◐ | ● | ○ |
| **Grades last verified** | 2026-09-11 | 2026-09-11 | 2026-09-11 | 2026-09-11 | 2026-09-11 |
| **Memory erasure docs re-checked** | 2026-09-21 | 2026-09-21 | unresolved (tutorial silent; primary page failed to load, 2026-09-21) | 2026-09-21 | not checked |

> **The Google column is the lowest-confidence assessment in this map.** Google has been renaming its agent products and recent documentation refers to an "Agent Platform SDK." Verify current naming, the native evaluation offering, and Memory Bank residency guarantees before citing any of it.

**How to read the shape of it.**

- **Anthropic bundles the intelligence layers** (memory curation, verification, orchestration) and is thinner on governance and residency.
- **AWS bundles infrastructure and governance** and stays model-neutral.
- **Microsoft bundles the most broadly** and adds something no one else has: distribution into Teams and Microsoft 365 Copilot, i.e. into software employees already have open.
- **OpenAI ships a library** and leaves hosting, verification, orchestration and governance to you.
- **The counter-trend worth watching** is portability tooling that defines an agent once as a neutral folder (prompt, skills, MCP server configs, subagents) and deploys it to several runtimes. The principle generalises past any particular tool: **own the definition, rent the runtime.**

---

## Table 2 — Scoped products and protocols

These entries address a defined subset of layers. The column states which layers they operate on; it is **not** a depth grade and should not be compared cell-for-cell with Table 1.

| Product | Layers addressed | Kind | Last verified |
|---|---|---|---|
| OpenCode | 1 Loop, 3 Context | Open-source coding harness (MIT) | 2026-09-11 |
| Hermes Agent | 4 Memory, 5 Verification, 7 Control plane | Open-source self-improving agent (MIT) | 2026-09-11 |
| OpenClaw 2.0 | 7 Control plane | Open-source gateway/harness (MIT) | 2026-09-01 |
| MCP | 2 Tools & integrations | Open protocol (moving MIT → Apache-2.0) | 2026-09-11 |
| Linear Agent | 2 Tools, 3 Context, 7 Control plane | Agent embedded in a SaaS product | 2026-09-16 |
| Taskade | 2 Tools, 3 Context, 4 Memory, 7 Control plane | No-code workspace + agent teams | 2026-09-16 |

---

## The platforms, one by one

> **A note on the "built for" reads.** Segment fit below is my inference from each product's capability profile and design choices, not vendor positioning. Vendors will describe their own audiences more broadly than this. I think the narrower read is more useful.

### Claude Managed Agents

**What it is.** Anthropic's managed agent runtime: a Claude Code-style harness run as a service, with memory, grading and multi-agent orchestration bundled in, accessed through the Claude API with beta headers. Primitives are agents (configuration), environments (where they run) and sessions (runs). Memories are files mounted into the agent's filesystem, permission-scoped, shareable across agents, each with an audit trail of which agent and session wrote it, plus rollback and redaction. "Outcomes" defines success as a rubric graded by a separate grader in its own context, and the agent iterates until it passes. "Dreaming" (research preview) is a scheduled review of past sessions and memory stores that finds recurring mistakes and shared preferences and restructures memory, with optional human review before changes land. Also: webhooks for event-driven triggers, vaults for credentials.

**Coverage.** Full on Loop, Context, Memory, Verification and Execution. Partial on Tools, Control plane and Governance.

**Built for** teams optimising for agent *quality* over portability and residency — organisations that have decided their differentiation is how good the agent gets at a repeating process, and are willing to accept a single-model dependency to get the strongest available memory-plus-verification bundle. It is the best-argued answer in the market to "how does this agent get better," and that is a real reason to choose it.

**Weakest at: governance and residency, which is the exact axis on which it is most likely to be blocked.** It is Claude-only, so there is no model neutrality. Residency was reported as US infrastructure only, but **that is third-party reporting from May 2026 and must be re-verified** — do not carry it into a client conversation unchecked. The deeper question is erasure across the dreaming boundary: `memories.delete` and `memory_versions.redact` work within a store, but dreams write to a separate output store, so erasing a memory in the source store does not reach a dream output that already absorbed it (Anthropic's memory and dreams docs, read 2026-09-21). Permission composition for out-of-band curation is also explicitly left to the builder: the stated approach is that you select which transcripts feed a dreaming job, mirroring your agents' permissions yourself, which becomes an access-control design problem at any real headcount. Dreaming remains a research preview.

### AWS AgentCore (Amazon Bedrock AgentCore)

**What it is.** A set of modular managed services (Runtime, Memory, Gateway, Identity, Policy, Observability) plus a managed harness. The harness (in preview as of 2026-09-11) lets you declare model, tools and instructions and have AWS run the loop; it is built on the open-source Strands Agents SDK, supports multiple model providers, and can switch models mid-session. Runtime gives agents written in any framework managed infrastructure with per-session microVM isolation. Memory covers short-term conversation plus long-term extracted insights and preferences with configurable extraction strategies. Gateway exposes existing APIs and serverless functions as MCP tools behind one authenticated endpoint. Policy and Identity enforce tool permissions outside agent code and manage agent and user credentials centrally. Classic Bedrock Agents is in maintenance mode per AWS documentation; AgentCore is the path forward.

**Coverage.** Full on Loop, Tools, Memory, Execution and Governance. Partial on Context, Verification and Control plane.

**Built for** EU-regulated enterprises and infrastructure-first teams that want model neutrality. EU regions including Frankfurt, GovCloud expansion announced August 2026, policy enforced outside agent code, and no model lock-in — that combination makes it the default managed candidate whenever residency and auditability are gating requirements rather than nice-to-haves.

**Weakest at: verification.** Layer 5 is graded partial, and that is conspicuous next to how complete the rest of the stack is. Check what evaluation tooling ships natively before assuming the improvement loop is covered; on current evidence you will be assembling it. Also confirm the managed harness's GA status, since it was still in preview at last verification.

### Microsoft Foundry Agent Service

**What it is.** The broadest bundle in this map. Hosted agents are containers serving the OpenAI `/responses` contract (GA around July 2026 — that date is approximate), declared in a manifest and deployed, with Foundry handling scaling, networking and lifecycle, and each session getting a hypervisor-isolated sandbox with a persistent filesystem. Memory (preview) offers procedural, user and session types built around extraction, consolidation, storage and retrieval. The Agent Optimizer does rubric-based evaluation and suggests fixes; tracing runs on OpenTelemetry and each eval result links back to the production trace that produced it, in the Foundry Control Plane. Toolboxes provide prebuilt capabilities, MCP integrations and skills. Routines and long-running agents cover scheduling, with publication into Teams and Microsoft 365 Copilot. The Agent Control Specification (ACS) standardises runtime policy, and Microsoft's memory-poisoning guidance places validation between model output and memory persistence. The companion open-source framework is Microsoft Agent Framework (MIT). Iberdrola is cited as a customer reference in energy operations.

**Coverage.** Full on every layer except Context (partial) — the only platform here with that profile.

**Built for** enterprises already standardised on Microsoft 365. The distribution advantage is the decisive factor and it is not a technical one: an agent that publishes into Teams and M365 Copilot lands in software employees already have open, which removes the adoption problem that kills most internal agent projects. If your organisation runs on Microsoft, the burden of proof sits on anyone proposing something else.

**Weakest at: preview-versus-GA clarity, and the credibility of its own headline numbers.** Memory was in preview at last verification, and several evaluation features need GA status confirmed before you quote them commercially. Microsoft reports Tau-bench gains of 7 to 14 points from procedural memory at near-baseline cost — **that is Microsoft's own reported figure, not an independent benchmark**, and should be presented as such. The other consideration is strategic rather than functional: this is the broadest bundle in the map, and breadth of bundle is exactly what creates switching cost.

### Google Vertex AI Agent Engine (with ADK)

**What it is.** Google's managed runtime for agents built with the Agent Development Kit, its open-source framework (Python, Go and others, Apache-2.0). ADK defines agents and a runner that connects them to session and memory services. Agent Engine Sessions store conversation history within a session. Memory Bank generates long-term memories from session events using a topic-based extraction method from Google Research (ACL 2025), consolidates them, and retrieves them scoped by user and app; default topics include explicit remember/forget instructions and custom topics are supported. **Memory generation is not automatic in ADK.** `add_session_to_memory` has to be triggered, for example from callbacks, and retrieval runs through memory tools the runner orchestrates. A2A, the agent-to-agent protocol, originated at Google.

**Coverage.** Full on Memory only; partial on all seven other layers.

**Built for** teams already committed to Google Cloud and Gemini whose primary requirement is long-term memory. The explicit memory-write trigger is a design virtue, not an omission: it makes the moment a fact becomes durable an auditable event rather than a side effect, which is the right default for anything with a retention obligation.

**Weakest at: being verifiable.** **This is the lowest-confidence entry in this map.** Product naming has been in flux, recent documentation refers to an "Agent Platform SDK," and I would verify three things before citing anything: current product naming and scope, the native evaluation offering, and residency guarantees for Memory Bank specifically. On the grading as it stands, everything outside Memory is partial, so it is the thinnest of the four hyperscaler bundles on the layers where differentiation now lives.

### OpenAI Agents SDK

**What it is.** OpenAI's open-source library for building agents (MIT; Python first, TypeScript following). Since the April 2026 update it ships a "model-native" harness — the loop OpenAI uses in Codex, packaged for developers. The harness is the control plane: instructions, tools, approvals, tracing, handoffs, resume bookkeeping. The sandbox is the execution plane, kept deliberately separate: filesystem, shell, packages, mounted storage and snapshots, running on your own infrastructure or a supported provider (Blaxel, Cloudflare, Daytona, E2B, Modal, Runloop, Vercel). The Manifest is a portable description of the agent's workspace (files, mounts, dependencies) so the same agent runs locally or on any supported provider. Memory is configurable: you control when memories are created and where they are stored. Code mode and subagents were in development as of April 2026. The Assistants API is being sunset in favour of the Responses API, targeted for mid-2026.

**Coverage.** Full on Loop. Partial on Tools, Context, Memory and Execution. **Nothing on Verification, Control plane or Governance.**

**Built for** product builders shipping an agent as part of their own software, who want the loop solved and everything else under their own control. It is a library, not a platform, and it is honest about that.

**Weakest at: everything above a single run.** No verification, no control plane, no governance — three zeroes, which for an enterprise deployment means three projects you own. Models are OpenAI-first. But read the Manifest design regardless of whether you adopt the SDK: it is the cleanest expression anywhere of the harness/sandbox split and of the "own the definition, rent the runtime" principle.

---

## The scoped products and the protocol

### MCP (Model Context Protocol) — the connective tissue, not a platform

**What it is.** The open protocol connecting agents to tools, data and prompts, and the de facto standard across every harness and platform in this map. A client inside the agent host connects to servers; servers expose tools (actions), resources (data) and prompts (templates). Transports: stdio for local servers, HTTP with OAuth-based authorization for remote ones. Licensing is moving from MIT to Apache-2.0 — new code and spec contributions Apache-2.0, non-spec docs CC-BY-4.0, older contributions remaining MIT until their authors consent to relicensing.

**Layer.** 2 only, and deliberately so. **It is not a platform and should not be compared to one.** Grading it against eight layers would be a category error; it is the seam that lets the other eleven products in this document interoperate at all.

**Built for** everyone, which is the point of a protocol. The managed platforms build gateways on top of it (AWS AgentCore Gateway, Microsoft Foundry Toolboxes); SaaS products expose their data through it; and, increasingly, SaaS products' own agents *consume* it in the other direction.

**Weakest at: authorization and per-tool permission scoping.** That is where agent security incidents cluster, and it is the area to watch as the specification matures. See the OWASP Top 10 for Agentic Applications (2026) for the current threat picture. My assessment: adopted, and the default for any tool integration.

### OpenCode

**What it is.** An open-source, provider-agnostic coding agent (MIT) — the open counterpart to Claude Code. Two primary agents switched by the user: `build` with full tool access, and `plan`, which is read-only and asks before changing anything. A `@general` subagent handles multi-step searches in its own context. Client/server split: the terminal UI is one client of a local server exposing an HTTP API, so the desktop app, IDE extension and your own scripts drive the same sessions. Project rules live in a committed `AGENTS.md`; long sessions are compacted. Widely adopted.

**Layers addressed.** 1 (Loop) and 3 (Context).

**Built for** developers who want a model-agnostic coding harness, and (the reason it is in this document) for anyone who wants to *read* a production agent loop. It is the most legible reference implementation available.

**Weakest at: memory, and it stops there deliberately.** There is no cross-session learning beyond the rules file and compaction. Study it for the loop, the plan/build split and context handling. Do not study it for memory.

### Hermes Agent

**What it is.** Nous Research's open-source self-improving general-purpose agent (MIT, Python, self-hosted), and the reference design for **procedural memory**: an agent that writes down *how* to do things, not just what it was told. Bounded declarative memory in two files, `MEMORY.md` (the agent's own notes) and `USER.md` (its model of the user), with size limits that force consolidation rather than accumulation. Procedural memory as skills: at intervals during a session the agent considers turning a successful workflow into a `SKILL.md` — the same portable format OpenClaw uses. Episodic recall by search over past sessions. Self-evolution via community pipelines using DSPy + GEPA to optimise skills and prompts against a metric. Several terminal backends (local, containers, remote and cloud sandboxes), a messaging gateway, and cron scheduling.

**Layers addressed.** 4 (Memory), 5 (Verification), 7 (Control plane).

**Built for** teams and individuals experimenting seriously with procedural memory — single-agent depth rather than orchestration breadth. It is the counter-design to OpenClaw: one agent that gets better, rather than many agents routed well. Because both use `SKILL.md`, skills move between them.

**Weakest at: the thing it is most famous for.** A known failure mode of the self-evolution loop is the agent gaming its own optimisation, which is why well-built setups add a separate verification job to block it. It is also the concrete case behind the "encoded mistakes that replay forever" risk on [Layer 4](layers/04-memory.md): memory and skills are written automatically, with no human gate equivalent to a code review. The open question I would want answered before adopting it anywhere consequential is whether procedural memory *measurably* improves outcomes or only accumulates — and that is an experiment, not a literature review.

### OpenClaw 2.0

**What it is.** A personal and team AI-agent harness (MIT, OpenClaw Foundation), self-hostable and cross-platform on a Node.js runtime. Layered: Gateway (channel adapters for Slack, WhatsApp, Discord and web chat, plus identity, session lifecycle and authorization) → Agent Runtime (reasoning, calling pluggable skills) → Skills → Storage (SQLite for sessions and transcripts as of 2.0, migrated from flat files). Procedural knowledge stays as git-diffable, human-editable `SKILL.md` files, with a shared skill registry. Authorization lives at the gateway/session layer: approvals bound to the exact request, command, session and person; per-session access tiers (read-only / guarded / workspace / full); a team-scoped secret store separating protected values from agent-readable ones. A small trusted kernel, with everything else as plugins. Version 2.0 (v2026.8.1, released 30 August 2026) added shared cloud sessions — multiple humans can join, leave or take over a running agent session while it keeps its context. Microsoft built its enterprise agent Scout on it, announced at Build 2026.

**Layer addressed.** 7 (Control plane) — with a governance model, at the gateway, that is worth copying precisely.

**Built for** technical self-hosters and teams who want to own their control plane: organisations for whom "state stays on infrastructure we control" is a requirement, and who have the engineering capacity to run it. It is the strongest available open template for the Gateway → Runtime → Skills → Storage shape and for what a genuinely shared session object needs to support.

**Weakest at: being a managed product.** You run it, you operate it, you secure it. The kernel/plugin split that keeps the trusted core small also means anything you build as a plugin lives outside that core's privileges by design — worth understanding before committing to it as a base, because it is a structural consequence of the architecture rather than a gap that will be closed. Note also that comparable "multiple humans in one live agent session" capability is thin across the market: the nearest competing products were early or in preview as of September 2026, and multi-agent orchestration for a single user is a different thing that is easily confused with it.

### Linear Agent

**What it is.** An AI agent built into Linear, the issue tracker and product-delivery tool, in public beta as of 2026-03-24 (confirmed against the changelog). It reads the workspace's roadmap, issues and customer requests to synthesise context, draft updates, triage, and create issues from discussions; extracts requirements and themes from feedback; gives project status and risk callouts. Available in-app, in issue comments, Slack and Microsoft Teams. Successful agent conversations can be saved as reusable **Skills**, invoked by menu or slash command — the same shape as the portable `SKILL.md` pattern, but scoped to Linear's workspace rather than portable across harnesses. Triage-triggered **Automations** are restricted to Business and Enterprise plans. **MCP support** (confirmed via a separate changelog, 2026-04-23): the agent connects to external MCP servers to pull outside context into drafts and updates — the changelog's own examples are Granola, Glean, Notion and PostHog — with admin control via allowlists and workspace-level MCP permissions.

**Layers addressed.** 2 (Tools), 3 (Context), 7 (Control plane).

**Built for** product and engineering teams already living in Linear, doing committed delivery work rather than open-ended research. It is not a general agent platform and does not try to be. Its structural interest to this map is as the clearest instance of MCP *consumption* (a SaaS product's own agent pulling in external context), which is the reverse of how MCP is usually discussed.

**Weakest at: being verifiable at the edges of its feature set, and portability of what it learns.** Two specific cautions. **Code Intelligence** (code-aware diagnosis and spec design, Business/Enterprise only) was listed as "coming soon" in the March changelog and its status was unconfirmed at last verification; re-check Linear's current documentation before relying on it. And **"Coding Sessions" and "Linear Diffs" could not be confirmed**: both appeared in a secondary research summary but on neither of the two changelog pages I checked. They may exist elsewhere in Linear's documentation, may be mis-attributed, or may have shipped after 2026-09-16. **Do not cite them without checking Linear's current docs directly.** On pricing: agent and Skills were included on all plans during beta, with the vendor stating chat is expected to stay included at GA and high-volume features possibly moving to usage-based pricing — not finalised. Architecturally, Skills are workspace-scoped, so the procedural memory you build up here does not travel.

### Taskade

**What it is.** A business-user-facing workspace bundling project management (projects, tasks, documents, mind maps) with several agent products on top: custom AI Agents; **AI Teams**, where multiple agents collaborate on a shared task via handoffs and peer review; AI Automations with event-triggered workflows across a large integration catalogue; and Taskade Genesis, a prompt-based builder for AI-powered apps and dashboards that stay connected to live workspace data. Persistence is three-layered: projects as structured, synced records (not chat transcripts) holding notes, tasks, custom fields, files and views; uploaded files, URLs and connected cloud storage indexed separately as searchable background knowledge; and workspace-scoped agent memory persisting across sessions independent of any project. A public API plus a hosted MCP server expose and can write to workspaces, projects, tasks, agents and media. Hosted SaaS across web, mobile and desktop; **no self-host option**.

**Layers addressed.** 2 (Tools), 3 (Context), 4 (Memory), 7 (Control plane).

**Built for** business users doing upstream discovery work (research, briefs, cross-functional coordination), not engineering teams. Its "AI Teams" is the gateway-first multi-agent pattern of [Layer 7](layers/07-control-plane.md) sold no-code to a non-engineering audience, which is worth tracking as evidence of how far down the org that pattern has spread rather than as an architecture to imitate. The clean operating split against a delivery tool like Linear is by stage: unstructured upstream work here, committed downstream work there.

**Weakest at: retrieval, with the vendor's own material conceding the point.** Taskade distinguishes its retrieval (agents searching indexed workspace content) from a dedicated enterprise-search or RAG platform, and publishes no evidence of hybrid keyword-plus-vector retrieval, re-ranking, or citation-level provenance. Read it as an integrated workspace-search experience, not a competitor to a purpose-built retrieval stack — and keep anything requiring auditable, citation-grade evidence in versioned files rather than trusting workspace search as the system of record.

**Two numbers to handle carefully.** Adoption figures confirmed against the source as of July 2026 (150,000+ apps built with Genesis, 535,000+ monthly active users, 10,000+ community app clones per month, reported as 5.4× growth versus January 2026) are **Taskade's own reporting with no independent audit; the source page itself labels them "original platform data."** And a figure that did **not** survive fact-checking: a secondary summary reported "63% of Genesis users are non-developers" as Taskade-specific, but the primary source states "63% of AI app builder users have no coding background" as a **category-wide** figure, not broken out for Taskade. I include that correction rather than dropping the number quietly, because the error is instructive about how vendor statistics propagate.

Retention, per Taskade's privacy policy: workspace content is retained until deleted or the account is closed; deletion may take up to 30 days and encrypted backups can persist longer. That is stated current practice, not a contractual retention guarantee.

---

## What the comparison actually shows

Three observations I would defend.

**Claude Managed Agents is the strongest platform at making an agent *good* — memory and verification are both full coverage. AWS AgentCore is the strongest at making an agent *safe to deploy* — governance is full coverage, but its verification is only partial.** That split isn't a gap either vendor forgot to close: Anthropic is optimizing for model quality, AWS for enterprise infrastructure and compliance. (Microsoft Foundry is the exception, bundling broadly enough to grade full on both — at the cost of more of its stack sitting in preview.) If you need both a self-improving agent and governance a regulator will accept, you are integrating vendors, not choosing one.

**Distribution is an underrated axis.** The hardest part of an internal agent deployment is usually not capability but adoption, and publishing into software people already have open is worth more than a feature. Only one platform in this comparison has that.

**The layers with no managed answer are the ones worth owning.** Context (Layer 3) is mostly implicit inside every runtime, verification (Layer 5) is bundled into the platforms that lead on it, though neutral products such as Braintrust exist, and integration into systems of record (Layer 2) is where the real cost sits and where nobody can do the work for you. That's not an accident: those three layers are where your proprietary knowledge actually lives, which is exactly why no vendor can sell it to you.
