# Layer 2 — Tools & integrations

## The problem

An agent that cannot touch your systems of record is a chatbot. The value of an agent is almost entirely a function of what it can read and what it can write — the CRM, the ticket tracker, the ERP, the data warehouse, the browser, the shell. Getting to those systems is where agent projects actually spend their time and money.

This is the least glamorous line in any proposal and reliably the largest. Integration into systems of record is usually both the biggest cost and the longest lead time in an agent project, and it is the line most often underestimated, because the demo was built against a mock. If you take one planning heuristic from this map, take this one: budget Layer 2 like an integration project, not like an AI project.

## Concepts worth owning

**MCP is the tool protocol.** The Model Context Protocol is the de facto standard across every harness and platform in this map, and the reasonable default for any new tool integration. A client inside the agent host connects to servers; servers expose *tools* (actions), *resources* (data) and *prompts* (templates). Transports are stdio for local servers and HTTP with OAuth-based authorization for remote ones. Licensing is moving from MIT to Apache-2.0 — new code and spec contributions are Apache-2.0, non-spec docs are CC-BY-4.0, and older contributions stay MIT until their authors consent to relicensing.

**The gateway pattern.** Rather than giving each agent its own credentials to each system, you put existing REST APIs, serverless functions and internal services behind a single authenticated MCP front door, with central auth and policy. AWS AgentCore Gateway and Microsoft Foundry Toolboxes both productise exactly this. The important consequence is not convenience — it is that tool permissions move out of agent code and into infrastructure, which is what makes [Layer 8](08-governance.md) enforceable.

**Skills versus tools.** A tool is a capability; a skill is a *method* — how to do a particular job with the tools available. The `SKILL.md` convention packages instructions plus scripts that an agent loads on demand, and it is portable across harnesses: OpenClaw and Hermes Agent both use it, and it is how procedural memory gets written down (see [Layer 4](04-memory.md)). Keeping the distinction sharp matters because the two have different owners, different review cycles, and different failure modes.

**Tool design is UX for models.** Fewer, well-named, well-described tools beat many thin wrappers around the same API. A tool surface with ninety near-identical endpoints is a context-window problem and a selection-error problem before it is anything else. On-demand tool discovery (searching and loading tools when relevant rather than declaring all of them up front) is the production answer at scale.

**Protocols, by seam.** MCP covers agent → tools. A2A (originated at Google) covers agent → agent. ACP covers editor → agent. When someone asks "which protocol should we adopt," the useful reply is "for which seam."

**MCP as consumption, not only exposure.** Nearly every treatment of MCP frames it as something *your* systems expose to an agent. The reverse direction is now real and underdiscussed: a SaaS product's own built-in agent consuming *external* MCP servers to pull outside context into its workspace. Linear Agent does this — its changelog cites Granola for meeting notes, Glean for enterprise search, Notion for interview notes, and PostHog for product data, with admin control via allowlists and workspace-level MCP permissions. If your systems of record are increasingly agent-bearing SaaS products, the integration question becomes bidirectional and the permissioning question gets harder.

**Search and research as a hosted API.** A separate build-vs-buy question from the gateway pattern: instead of exposing your own systems as tools, you rent somebody else's retrieval, ranking and synthesis wholesale. This is at least a two-vendor category now (You.com and Perplexity's Sonar API among them). Treat published comparisons with care — vendor-reported benchmark numbers in this category currently have no shared methodology to compare them by, so cross-vendor claims are not commensurable.

## The landscape

Open source is dominated by the MCP ecosystem itself: the protocol specification, SDKs in many languages, and a large and uneven population of community servers. The `SKILL.md` format covers the packaging side.

On the managed side, AWS AgentCore is graded full coverage on this layer, largely on the strength of Gateway plus centrally managed agent and user credentials; Microsoft Foundry is also full, via Toolboxes (prebuilt capabilities, MCP integrations and skills). Claude Managed Agents, Google Vertex Agent Engine and the OpenAI Agents SDK are each graded partial. See the [comparison](../framework-comparison.md) for the full matrix and its provenance.

## The business-case questions

Which systems of record must the agent **read**? Which must it **write**? And the question that decides your audit story: do credentials belong to the agent itself, to the user via OAuth pass-through, or to a service account? Each of those three answers produces a different liability position, and the choice belongs to [Layer 8](08-governance.md), but it is forced here.
