# Layer 8 — Governance

## The problem

Who is the agent, what may it do, who can see what it did, how do you undo it, and where does its data live? These questions are boring right up until the moment they are the only questions anyone is asking, which is usually the moment a deal reaches procurement or an agent does something expensive.

Governance is the layer that determines whether an agent system can be deployed at all in a regulated environment, and it is the layer most often deferred to "phase two." That deferral is a mistake for a specific structural reason: several of these controls (identity model, policy placement, audit provenance) are architectural. They are cheap to build in and very expensive to retrofit.

## Concepts worth owning

**Agent identity.** The agent acts as itself, as the user (OAuth pass-through), or as a service account. Three different audit stories and three different liability positions. Acting as the user gives you clean attribution and inherits the user's permissions, which is usually right and occasionally catastrophic — a user with broad standing access grants an agent the same. A service account gives you a clean permission boundary and a muddy attribution story. Choose deliberately; do not let the choice be made by whichever SDK example you copied.

**Policy outside agent code.** Enforce tool permissions at the gateway or the runtime, never in the prompt. A prompt instruction not to do something is a request; a gateway rule is a control. AWS AgentCore Policy and Microsoft's Agent Control Specification (ACS) both work this way, and it is the single clearest dividing line between platforms that are enterprise-ready on this layer and platforms that are not. The same principle applies to network egress: allowlist at the network layer, not by asking the model nicely.

**Guardrail placement.** Three checkpoints, each with a tripwire that halts the run:

| Checkpoint | Fires | Catches |
|---|---|---|
| Input | Before the model sees it | Injection in incoming content, out-of-scope requests |
| Output | Before a person or downstream system sees it | Leaked secrets, policy-violating content |
| Tool call | Before the action executes | Over-privileged or destructive actions |

All three belong here rather than in [Layer 5](05-verification.md), because they *block* rather than *measure*. Evals and guardrails are constantly conflated and they have different owners.

**Audit and provenance.** Every action and every memory write traceable to an agent, a session and an input. The memory half is the part that is newly hard and frequently missing — Claude Managed Agents gives each memory an audit trail recording which agent and session wrote it, with rollback and redaction, which is the right shape.

**Rollback and redaction.** Undo a memory, redact content from history, reverse an action. Ask vendors about all three separately; they are commonly conflated, and the answers differ.

**Residency.** Varies sharply by platform and changes often. AWS AgentCore offers EU regions including Frankfurt, with GovCloud expansion announced in August 2026, which makes it the most likely managed choice for EU-regulated work as of this writing. Google Vertex Agent Engine offers EU regions, though Memory Bank residency guarantees specifically are one of the things I would verify first about that platform. Claude Managed Agents was reported as US infrastructure only — **that is third-party reporting from May 2026 and needs re-verifying before any EU conversation**. Treat every residency statement in this repo, including that one, as needing confirmation against current vendor documentation before it goes in front of a client.

**The erasure problem, stated properly.** GDPR-style erasure obligations are well understood for records, and as of 2026-09-21 three of the four platforms with a memory product document record-level deletion that reaches consolidated memory: Claude Managed Agents, AWS AgentCore and Microsoft Foundry. Foundry's is still preview and Google's is undocumented; details are in [Layer 4](04-memory.md). The unsolved part is *derived* artefacts, where erasure has to cross a boundary. Claude's dreams write a separate output store that deletion in the source store does not reach, and I could not confirm from AWS's docs whether a summary goes when its source record does. A skill synthesised from a person's data has the same shape. No platform documents erasure across such a boundary, and the public talk and write-up on dreaming did not address erasure at all. Related and equally unresolved: **how permissions compose for out-of-band curation.** The stated approach for dreaming is that you choose which transcripts feed the job, thereby mirroring your agents' permissions yourself. With hundreds of users and heterogeneous permission sets, that becomes an access-control design problem that no platform appears to solve out of the box.

**Threat model.** The OWASP Top 10 for Agentic Applications (2026) is the right starting list: prompt injection via tool results, memory poisoning, over-privileged tools, and the rest. Note how many of the top risks are *not* model risks — they are permission and data-flow risks, which is to say they are solved on this layer and [Layer 6](06-execution-environment.md), not by a better model.

**Observability standards.** OpenTelemetry's GenAI semantic conventions are the emerging vendor-neutral answer for traces, and Microsoft's tracing is built on them. Standard traces are what make the "jump from an eval score to the production trace that caused it" property portable rather than proprietary.

## The landscape

Open source here is standards and checklists rather than products: OpenTelemetry GenAI conventions for observability, the OWASP Agentic Top 10 for threat modelling. The enforcement machinery is mostly in the platforms, or in whatever gateway you run — OpenClaw's model is worth studying as the self-hosted reference, with approvals bound to the exact request, command, session and person, per-session access tiers (read-only / guarded / workspace / full), and a team-scoped secret store separating protected values from agent-readable ones.

On the managed side this is the layer with the sharpest split. **AWS AgentCore and Microsoft Foundry are both graded full** — Policy and Identity in AWS's case, ACS and the Foundry Control Plane in Microsoft's. **Claude Managed Agents is graded partial**, with vaults and memory audit, but residency unconfirmed and erasure documented only within a single store (see above). **Google Vertex Agent Engine is graded partial** and is the lowest-confidence assessment in the map. Put plainly: the vendor best at making an agent *improve* (Claude Managed Agents, full on memory and verification) is not the vendor best at making that agent *deployable under audit* (AWS and Microsoft, full on governance). Pick which axis you're buying, and plan to build the other yourself.

## The business-case questions

What does a wrong action cost? Where must the data live? Can a person's data be erased from the agent's memory, including anything derived from it, and can you *prove* it?
