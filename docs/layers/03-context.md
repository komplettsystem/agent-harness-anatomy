# Layer 3 — Context

## The problem

The model's context window is a scarce, expensive resource, and every step of the loop forces the same decision: of everything that could be known, what goes in *this* call? Instructions, retrieved documents, the conversation so far, the results of the last twelve tool calls — all of it competes for the same budget, and the wrong selection degrades output quality in ways that look like model failure but are not.

This layer is where most agent quality problems actually live. It also carries the hardest question in any non-coding agent project, which is not technical: *where does the knowledge required to do this job currently exist?* Very often the honest answer is "in three people's heads," and that is the blocker, not the harness.

## Concepts worth owning

**Compaction.** Summarise older turns to stay under the limit. It is lossy by design, so the only interesting question is what gets dropped and whether the agent can tell that something was. Compaction that silently discards a constraint stated in turn three is a correctness bug wearing a performance costume.

**Compacting without full replay.** NVIDIA Labs' SoL-Pi (see [Layer 1](01-loop.md) for what it is) has two mechanisms here. Online Context Compact compresses completed subtasks at semantic boundaries as a trajectory runs, instead of replaying full history each time compaction triggers — a sharper version of the compaction concept above. ObservationPack replaces repeated large tool outputs with stable handles and paged recall, rather than re-emitting the same payload every time it's referenced.

**Subagents as context hygiene.** Delegate a messy, high-volume search to a subagent with a clean window, and take back only the answer. This is the cheapest available fix for tool-result pollution, and it is why "should we go multi-agent?" sometimes has a yes answer for reasons that have nothing to do with parallelism (see [Layer 7](07-control-plane.md)).

**Progressive disclosure.** Load skills and documents when they become relevant, not up front. The same principle applies to tool definitions. A harness that front-loads everything is trading context budget for implementation simplicity, and at scale that trade goes bad.

**Rules files.** `AGENTS.md` and `CLAUDE.md` are static, human-curated context: the project's conventions, committed to the repo, versioned like code. They are the most basic form of memory, and it is worth noticing that most coding harnesses stop here — a rules file plus compaction, with nothing that learns. That is a perfectly defensible design and it is also the ceiling on what those tools can do for you.

**Context stacked by rate of change.** The most useful production pattern I have seen for organising this layer is to order context by how fast each kind changes, from most stable to most volatile. LangChain's publicly documented paid-media agent separates: system prompt (role and navigation) → skills (folders of instructions, progressively disclosed) → wiki (domain pages on how the business area works) → live tools (spend, settings, pipeline — changing daily) → code (calculations, date windows, hard safeguards). Ordering it this way isn't just tidy: each tier has a different owner, a different review cadence, and a different correct storage medium. Stable knowledge belongs in versioned files that the agent searches; volatile knowledge belongs behind a tool call.

**Context that lives outside the repo.** For coding agents, context is mostly the repository. For every other business process it is not. Product, design and commercial context sits in Figma, Notion, a ticket tracker, or the CRM, and it evolves independently of anything you version. Both obvious approaches fail: baking it into skills makes it stale, and pulling it live over MCP makes evaluation unstable, because the source keeps changing and never reaches a "done" state against which you could score anything. The argument I find most convincing (made publicly by Tessl, under the heading "harness engineering beyond code") is that the harness needs an explicit intermediary layer maintaining that bridge, much as design-system teams have maintained the bridge between design and code by hand for years. For non-coding processes this is the normal case, not the edge case, and I do not think anyone has shipped a good general answer yet.

**Four kinds of persistent context, kept apart.** Worth separating explicitly when you design storage, because vendors blur them and they have different lifecycles:

| Kind | What it holds | Shape |
|---|---|---|
| State | Session and task state, tied to a session's lifecycle | Ephemeral, point-access |
| Context | The shared, slower-changing corpus: docs, code, decisions | Large, read-heavy, needs efficient retrieval |
| Episodic | Who decided what, when — across humans and agents | Outlives any session; needed for traceability and continuity |
| Procedural | How to do the work | Small, stable, git-versioned |

Locality (local-first versus cloud-managed) is a *separate axis* from memory type. Do not couple the two: "we need this on-premises" is an infrastructure answer, not an architecture one. State, episodic and procedural are treated in [Layer 4](04-memory.md); retrieval over the context corpus is this layer's job.

**Hybrid retrieval.** The practical open-source answer for the context corpus is hybrid: literal/regex match, BM25 keyword search, and vector similarity behind one interface, with the agent choosing a route. Local-first implementations exist that index a workspace once and run embeddings on-device by default — `zg` (zvec-grep, Qwen Developers, Apache-2.0) is a current example, and its narrow MCP surface reflects a deliberate design choice: it exposes search tools only, keeping index lifecycle with the CLI so an agent never silently reindexes. Vendor A/B benchmarks report meaningful cuts in tool calls and input tokens versus naive search, though those samples are small and unreplicated, so treat the magnitude as directional rather than as a measured result. What retrieval does *not* solve is curation, staleness, or access control — those stay with your process and version-control layer, and they are the parts that get expensive as a corpus grows across a team.

## The landscape

This layer is unusual in that the managed platforms mostly do *not* expose it as a product. Context handling is implicit inside each runtime: you get whatever compaction and retrieval policy the vendor chose, with limited visibility. The exception in the grading is Claude Managed Agents, the only platform graded full coverage on Layer 3; AWS AgentCore, Microsoft Foundry, Google Vertex Agent Engine and the OpenAI Agents SDK are all graded partial. Open source is where the observable designs are: OpenCode for compaction and subagent delegation, LangChain Deep Agents for the layered context stack, and the hybrid-search tools for the retrieval half.

## The business-case question

**What does a person doing this job need to know, and where does that live today** — wiki, heads, email, ERP, a spreadsheet someone maintains by hand? If the answer is "heads," the first deliverable is not an agent. It is writing the knowledge down, and that is a real project with a real timeline that should appear in the plan as such.
