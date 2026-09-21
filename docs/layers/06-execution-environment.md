# Layer 6 — Execution environment

## The problem

An agent's actions run somewhere: a filesystem, a shell, installed packages, a network, a set of credentials. That "somewhere" determines the blast radius of every mistake and every successful prompt injection. The question this layer answers is not "where does the code run" but "what is the worst thing that can happen when the agent is wrong, or when someone else is driving it."

It is also, less dramatically, the layer that determines whether a long-running agent can be interrupted and resumed, which is an operational property you will care about the first time a forty-minute run dies at minute thirty-eight.

## Concepts worth owning

**Per-session isolation.** A microVM or VM per session, not a shared container. Both AWS and Microsoft moved to hypervisor-level isolation for their managed offerings, which is a reasonable signal about where the bar now sits. Shared-container isolation between sessions is an acceptable trade only when every session is equally trusted, and in a multi-tenant or multi-user deployment that is rarely true.

**Persistent workspace versus ephemeral sandbox.** Long-running agents need resumable state — snapshots, mounted storage, a filesystem that survives a restart. One-shot tasks do not, and paying for persistence you do not need buys you a state-management problem for free. Microsoft Foundry gives each session a hypervisor-isolated sandbox with a persistent filesystem; LangChain's published reference build gives each chat thread its own sandbox and checkpoint. Both are the persistent end of the spectrum.

**Portable workspace definitions.** OpenAI's Manifest abstraction describes a workspace once (files, mounts, dependencies) and runs it on any supported sandbox provider. This is the clearest expression anywhere of a principle I would apply across the whole map: **own the definition, rent the runtime.** It is worth reading even if you never adopt the SDK, because the same idea applied to skills, MCP server configs and prompts is what keeps a whole agent portable.

**Two streams of risk, not one.** A useful decomposition from the enterprise coding-agent vendors: an agent session actually contains two flows with different risk profiles. Model calls send context *out*: that is a disclosure risk. Tool calls execute actions *in* your systems: that is a blast-radius risk. They want different controls. Network-layer egress allowlisting handles the first and belongs in infrastructure, not in the prompt; sandboxing and permission scoping handle the second. Treating them as one undifferentiated "agent security" concern gets one of them wrong.

**Credential vaults.** Secrets injected at runtime, never placed in the prompt, never written into the memory store. The memory-store half of that sentence is the one people forget, and it is the one that turns a credential mistake into a permanent one. Claude Managed Agents ships vaults for exactly this; a team-scoped secret store that separates protected values from agent-readable ones (OpenClaw's design) is the self-hosted equivalent.

## The landscape

**Open source and independent providers.** E2B and Daytona are the open-source sandboxes. Modal, Cloudflare, Vercel, Blaxel and Runloop are hosted options. The OpenAI Agents SDK supports all of these as pluggable backends, which makes it the most convenient place to see the abstraction boundary done cleanly. Self-hosted harnesses generally offer a range of terminal backends (local, container, remote SSH, and cloud sandbox) as a configuration choice rather than an architecture.

**Managed.** AWS AgentCore Runtime provides per-session microVM isolation and is graded full on this layer; Microsoft Foundry provides per-session VM-isolated sandboxes with persistent filesystems and is also graded full; Claude Managed Agents exposes "environments" as a first-class primitive alongside agents and sessions, and is graded full. Google Vertex Agent Engine and the OpenAI Agents SDK are graded partial — in OpenAI's case because the SDK deliberately does not host anything, it only abstracts over providers who do.

## The business-case question

**What can the agent touch, and what is the worst thing it can do with that access?** Write the answer down as a sentence a non-engineer would recognise as a description of a bad day. If that sentence is alarming, the fix is usually here or in [Layer 8](08-governance.md), not in better prompting.
