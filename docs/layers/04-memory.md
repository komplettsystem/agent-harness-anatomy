# Layer 4 — Memory

## The problem

Without memory, every session starts from zero. The agent relearns the same conventions, repeats the same clarifying questions, rediscovers the same quirk in the same portal, and makes the same mistake it made yesterday. For a one-off task that is fine. For a process that runs two hundred times a month it is the whole cost case.

Memory is anything that survives the session and changes future behaviour. It is also the layer where the most damage gets done, because a bad memory does not fail once — it becomes a fact or a method that every future session reuses. **Memory without a verifier compounds errors as fast as it compounds skill.** That sentence is the reason [Layer 5](05-verification.md) exists and the reason this layer should never be bought on feature list alone.

## Concepts worth owning

**Three kinds. Keep them separate; vendors blur them.**

| Kind | Content | Example | Risk |
|---|---|---|---|
| Declarative (semantic) | Facts, preferences, conventions | "This client invoices in EUR, net 60" | Stale facts |
| Episodic | What happened in past sessions | Searchable session history | Privacy; retention obligations |
| Procedural | How to do the work | A skill written after a successful run | **Encoded mistakes that replay forever** |

They differ in retention policy, in who should curate them, in whether they can be reviewed by a human at all, and, critically, in how hard they are to erase. Any product that offers you "memory" as a single undifferentiated feature is making that distinction your problem.

**Write path versus read path.** Extraction → consolidation → storage → retrieval. Most failures happen at extraction and consolidation, not retrieval, which is the opposite of where most attention goes. Teams debug their vector search while the actual bug is that the agent wrote down a hypothesis as a fact three sessions ago.

**In-band versus out-of-band.** In-band memory is read and written by the agent during the session. It pays off from the very next run, and it has two structural limits: the agent splits its effort between doing the task and curating for the future, and it cannot see patterns across sessions or across a fleet. Out-of-band curation runs separately, on its own budget, with visibility across many transcripts. Both are legitimate; only the second can catch a failure mode that appears once per hundred sessions.

**Curation.** Automatic (in-band), scheduled (out-of-band), or human-reviewed. Anthropic's "dreaming" in Claude Managed Agents is the productised out-of-band version. I classify it in this map as a **Layer 5 verifier rather than Layer 4 curation**, because what it actually does is *judge past behaviour* — memory restructuring is the output, not the mechanism. That reclassification matters for scoping: if you buy it, you are buying an evaluation capability, and you should hold it to an evaluation capability's standards. It is a research preview as of 2026-09-11.

**Scope is access control.** Org-wide read-only, team read/write, per-user, per-agent scratchpad. Memory is an access-control problem at least as much as a retrieval problem, and it is usually designed as though it were only the latter.

**Memory poisoning.** A bad extraction becomes a durable fact or a reusable skill. The defence goes *between model output and persistence* (validation, provenance, classifiers, audit), not after the fact. Microsoft's published memory-poisoning guidance for Foundry puts validation at exactly that seam, which is the right place.

**Reflected profiles.** A pattern now appearing in vendor tutorials: episodic memory feeding a derived profile that a periodic reflection rewrites — a writing-style profile updated from published output, say. Elegant, and without a verifier it can drift and then keep the drift, with no signal that anything went wrong.

**Memory as files.** The clearest convergence of 2026: memories stored as files on a filesystem. Hermes Agent uses a bounded `MEMORY.md` and `USER.md`; Claude Managed Agents mounts memory stores into the agent's filesystem. Files are inspectable, exportable, diffable and versionable; agents already search files competently with shell tools, so bespoke memory APIs buy less than they appear to; and search over an indexed store doubles as progressive disclosure. Bounding the file size is itself a design feature — it forces consolidation instead of endless accumulation.

**One narrow storage contract, shared.** A pattern worth copying regardless of backend, visible in AWS's Strands DynamoDB storage layer: rather than a bespoke store per stateful subsystem, define a single narrow contract (write / read / delete / list) and have session management, memory, transcripts and oversized-tool-result offloading all speak through it. One contract to secure, one to test, one to swap when the backend decision changes. Note the caveat AWS states explicitly about its own implementation: multi-tenant key prefixing is *query* scoping, not an *authorization* boundary. A principal with the right IAM permission can read across partitions. Tenant isolation has to live in the identity and application layer, never in the storage layer: a mistake that is very cheap to make and very expensive to discover.

**Production principles for shared memory.** From Anthropic's Applied AI team (AI Native DevCon London, 2026), these are the harness-side guardrails once many agents share one store over a long period. I read this table as a requirements list, not a description:

| Principle | Mechanism | Failure it prevents |
|---|---|---|
| Versioning | Every change stored with its source session or transcript and its author (agent or human); rollback possible | Bad updates that cannot be traced or undone |
| Concurrency | Optimistic check: hash before drafting and again before writing; on mismatch, re-read, redraft, retry | Agents silently overwriting each other |
| Permissioning | Tiers from org-wide knowledge (read-only for most agents), through team scopes, down to an agent's own scratchpad (writable) | One confused agent corrupting the context every other agent reads |
| Portability | A clean API, so curated memory can be used by other products and runtimes | Curation effort locked into one vendor |

Staleness and deliberate poisoning via prompt injection are the two risks these principles do *not* cover. That residual gap is precisely what out-of-band curation is for.

## The landscape

**Open source.**

| Project | License | What to look at |
|---|---|---|
| Hermes Agent (Nous Research) | MIT, Python, self-hosted | The reference design for procedural memory: an agent that writes down *how* to do things, not just what it was told. Bounded `MEMORY.md` / `USER.md`; agent-authored `SKILL.md` files; episodic recall by search over past sessions; a self-evolution pipeline. |
| Letta (formerly MemGPT) | Server | Agent-managed memory blocks; the original "LLM as operating system" memory hierarchy. |
| Mem0 | Library or service | Extraction and consolidation treated as a standalone layer rather than a runtime feature. |
| Graphiti (Zep) | Library | Temporal knowledge graph: facts carrying validity windows, which is the cleanest available answer to stale declarative memory. |

**Managed — four distinct designs.**

- **Memory as audited files, plus scheduled curation.** Claude Managed Agents: memories are files mounted into the agent's filesystem, permissions scoped, stores shareable across agents, each memory carrying an audit trail of which agent and session wrote it, with rollback and redaction. Dreaming adds scheduled review. Graded full coverage on this layer.
- **Short-term plus long-term with configurable extraction.** AWS AgentCore Memory: conversation memory plus extracted insights and preferences, with configurable extraction strategies. Graded full.
- **Topic-based extraction, explicitly triggered.** Google Vertex AI Agent Engine: Agent Engine Sessions hold within-session conversation state; Memory Bank generates long-term memories from session events using a topic-based extraction method from Google Research (ACL 2025), consolidates them, and retrieves scoped by user and app. Notably, **memory generation is not automatic in ADK** — `add_session_to_memory` has to be triggered, for example from a callback. Graded full on memory, and the only layer where Google grades full.
- **Typed memory: procedural, user, session.** Microsoft Foundry: built around extraction, consolidation, storage and retrieval. Microsoft reports Tau-bench gains of 7 to 14 points from procedural memory at near-baseline cost — **that is Microsoft's own reported figure, not an independent benchmark**, and memory was in preview as of 2026-09-11, so confirm GA status before quoting it to anyone.

Keep two things in mind on every memory conversation. The **Google entry is the lowest-confidence entry in this map** — Google has been renaming its agent products, recent documentation refers to an "Agent Platform SDK," and current naming, native evaluation offering and Memory Bank residency guarantees all need verifying before you cite any of it. And **erasure into consolidated memory is an unresolved question across the board.** For Claude Managed Agents specifically, whether redaction reaches memories that dreaming has already consolidated was not established in the sources I checked, and the public discussion of dreaming did not address erasure at all. The general form of the problem is worse than the specific one: once a *skill* has been derived from a person's data, deleting the source does not delete what was learned from it.

## The business-case questions

Does the process repeat often enough that a learned method pays for the cost of curating it? Who curates what the agent learns, and on what cadence? What are your retention and erasure obligations, and can you demonstrate compliance with them for derived and consolidated memory rather than just for raw records?
