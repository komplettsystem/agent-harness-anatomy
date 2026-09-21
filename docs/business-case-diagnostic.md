# Business-case diagnostic

A short structured way to decide whether a given process is a candidate for an agent, what kind of agent it wants, and how much autonomy it can safely be given. Five questions, a grid, and a ladder.

The purpose of this is to move the conversation off "which platform should we buy" (the wrong first question) and onto the properties of the *process*, which determine everything downstream, including which platform.

---

## Five questions

Score each Low / Medium / High.

| # | Question | What it decides |
|---|---|---|
| **Q1** | How cheaply and reliably can the output be checked? | The autonomy ceiling; and whether memory helps or poisons |
| **Q2** | How often does the process repeat? | Whether procedural memory pays for its curation cost |
| **Q3** | What does a wrong action cost? | Where humans sit on the autonomy ladder |
| **Q4** | Which systems of record does it touch, with what access? | Integration cost and timeline — usually the largest line in the plan |
| **Q5** | Where must the data live, and who can erase it? | The vendor shortlist; the governance design |

Notes on scoring each of them honestly:

**Q1 is the question.** Everything else is downstream. "The output is checked by the person who requested it, eventually, if they notice" is a Low, not a Medium. Be strict here: the whole value of the diagnostic comes from not flattering this answer. If Q1 is genuinely Low and cannot be raised, the first deliverable of the project is a verifier (a rubric, a deterministic check, a sampling process), and the agent comes second. See [Layer 5](layers/05-verification.md) for the full verifier ladder.

**Q2 is about the unit of repetition, and it is easy to get wrong.** A process can be low-repetition per user and high-repetition across users, and that distinction changes what memory is for: the shared, reusable knowledge is the *method*, not the case. Scope the memory accordingly.

**Q3 is rarely uniform.** Most real processes contain a common case with a low cost of error and a rare subclass with a high one. Score both. A single average score here produces a design that is simultaneously too cautious for the bulk of the work and too permissive for the part that matters.

**Q4 is where the money goes.** Integration with systems of record is typically the largest cost and the longest lead time in an agent project, and it is the line most often underestimated because the prototype ran against a mock. Budget it like an integration project. Answer the sub-question explicitly too: do credentials belong to the agent, to the user via OAuth pass-through, or to a service account? See [Layer 2](layers/02-tools-and-integrations.md).

**Q5 shortens the vendor list faster than any feature requirement.** Residency varies sharply by platform and changes often, so verify it against current documentation rather than against anything written down — including this repo. The harder half is erasure: can a person's data be removed not just from records but from *derived* artefacts (consolidated memories and learned skills), and can you prove it? See [Layer 8](layers/08-governance.md).

---

## Archetypes, from Q1 × Q2

Verifiability against repetition produces four distinct designs. They are not points on a maturity scale; they are different systems.

| | **Low repetition** | **High repetition** |
|---|---|---|
| **High verifiability** | **Task agent.** Autonomous, little or no memory. There is nothing worth learning because it will not come round again — invest in the tools, not the memory. | **Learning agent.** Autonomous, with procedural memory. This is the coding-like case, and the one that justifies the full stack. |
| **Low verifiability** | **Copilot.** The human drives; the agent drafts and suggests. Skip memory entirely — without a verifier it will compound whatever it got wrong. | **Verifier first.** Build a rubric grader plus sampled human review *before* granting any autonomy. Memory stays human-curated until the verifier is trusted. |

The bottom-right cell is the one that gets misdiagnosed most often, because high repetition looks like a strong business case and the low verifiability gets discovered later. High repetition with a weak verifier is the configuration that produces an agent which reliably and efficiently does the wrong thing at scale, and improves at doing it.

---

## The autonomy ladder

Set by Q3 (cost of error), **capped by Q1** (verifiability).

1. **Suggest** — the agent proposes; a human does the work.
2. **Draft** — the agent produces the artefact; a human reviews and submits.
3. **Act with approval at the action boundary** — the agent does everything up to the irreversible step, then waits. This is the workhorse rung.
4. **Act and report** — the agent acts, then tells someone what it did.
5. **Act silently** — the agent acts; only aggregates are reviewed.

**Rule of thumb: never climb above the level your verifier can support.** High cost of error plus a weak verifier means rung 2, whatever the demo showed. And climbing the ladder is not a maturity goal — plenty of well-designed systems should sit at rung 3 permanently, because that is where the economics are.

Two refinements worth applying. First, **the rung is per action class, not per agent.** Reading, drafting and filing are different risks and can sit on different rungs inside one process. Second, **the approval step is only real if the approver can actually evaluate it.** An approval card that presents a change no reviewer has the context to judge is theatre that transfers liability without adding safety; the design work is in making the boundary reviewable.

---

## Worked example (hypothetical): customer-support ticket triage

A generic illustration, not a real deployment. An agent reads inbound support tickets and routes them to the right queue with a priority and a suggested category.

| Q | Score | Reasoning |
|---|---|---|
| Q1 Verifiability | Medium | A cheap delayed proxy exists: if the receiving team reassigns the ticket, the routing was wrong. That is a deterministic signal available within hours, without a grader — but it measures routing, not priority, so priority needs a rubric or sampling. |
| Q2 Repetition | High | Thousands per month; the method generalises across tickets even though each ticket is new. |
| Q3 Cost of error | **Split.** Low for the common case, High for a rare subclass | A misrouted billing question costs a few hours of delay. A missed security disclosure or a regulatory complaint costs considerably more. |
| Q4 Systems | Ticketing system read *and* write; customer record read | One write integration and one read integration — small by the standards of Q4, which is why this is a good first project. |
| Q5 Data | Customer PII in ticket bodies; possibly special-category data in incoming text the agent does not control | Residency constrains the vendor list. Memory must hold *routing method*, never ticket contents. |

**The design that falls out.** Archetype: *learning agent* (high repetition, adequate verifiability, procedural memory pays off). But Q3 is split, so **the autonomy rung is not uniform**: rung 4 (act and report) for the common case, with the reassignment rate as the standing verifier, and rung 2 or 3 for the high-cost subclass, gated by a deterministic pre-filter that detects it and refuses to auto-route. Memory is scoped to routing heuristics (which symptoms belong to which queue) and explicitly never to ticket contents, which keeps the erasure obligation on the ticketing system where it already lives and out of the agent's memory store entirely.

The non-obvious output is the split rung. A single autonomy level for the whole process would be either too slow for 95% of tickets or too dangerous for the other 5%. That is the most common thing this diagnostic surfaces, and it is usually invisible until Q3 is scored twice.

---

## Using the results

The diagnostic produces four things worth writing down: a process portfolio plotted on the archetype grid, a buy-versus-build decision per layer, a verifier specification, and an autonomy roll-out plan with the rung for each action class.

The build-versus-buy pattern that follows from the map as a whole is fairly stable across cases. Rent [Layer 1](layers/01-loop.md) and [Layer 6](layers/06-execution-environment.md) — they are commodities. Rent [Layer 7](layers/07-control-plane.md) and [Layer 8](layers/08-governance.md) unless regulation forces your hand, in which case buy carefully against Q5. **Own [Layer 2](layers/02-tools-and-integrations.md) (proprietary integrations), [Layer 4](layers/04-memory.md) (process memory), and [Layer 5](layers/05-verification.md) (verifiers)**: those hold your knowledge, and they are what remains yours when you change platforms.

Keep the agent definition portable: skills, MCP server configurations and prompts as files in version control. **Own the definition, rent the runtime.**

And measure the right things. Verifier pass rate and human override rate tell you whether the system works. "Agent usage" tells you nothing at all.
