# Reference: Harness Principles

Oct 19, 2026 · @Rene Ravenel

Target for the Harness ISA project: the concerns any LLM agent harness must handle, and how today's harnesses handle them.

## What a harness is

A harness is everything around the model that makes it useful: the common formulation, from LangChain's Vivek Trivedy in March 2026, is **Agent = Model + Harness**. A model takes data in and emits text; it cannot keep durable state, run code, reach current knowledge, or set up environments. The harness supplies all of that.

Two framings from 2026 surveys are useful as checklists:

- **ETCLOVG** (TMLR submission): four structural pillars (Execution, Tooling, Context, Lifecycle) for what an agent can do, plus a control plane (Observability, Verification, Governance) for how safely it does it.
- **H = (E, T, C, S, L, V)** (Meng et al.): execution loop, tool registry, context manager, state store, lifecycle hooks, evaluation interface. Of 23 systems evaluated, only Claude Code, PRISM/OpenClaw, AIOS, OpenHands, and SWE-agent implement all six.

One more idea matters for an ISA: an explicit **authority hierarchy**. Content is labeled by authority, from provider policy down through organization, developer, project, user task, runtime reminders, tool observations, and finally untrusted retrieved content. Instructions found in lower layers are data, not policy, and approvals, credentials, and authorization stay outside model-directed compute.

## Harness concerns catalog

Eighteen concerns recur across the sources. The numbering is used by the coverage matrix below.

| # | Concern | What it involves | Where it is documented |
| --- | --- | --- | --- |
| 1 | Agent loop and control flow | Decide next step, act, observe, repeat; when to stop; who owns the loop | 12-Factor Agents (own your control flow); ReAct |
| 2 | Prompt and context assembly | What tokens go into each call, in what order, from which sources | 12-Factor (own your prompts, own your context window); Anthropic context engineering |
| 3 | Output decoding and validation | Parse model output, detect tool calls, check JSON and arguments, repair or reject | 12-Factor (tools are just structured outputs); provider structured outputs |
| 4 | Tool registry and dispatch | Declaring tools, schemas, discovery, progressive disclosure, list changes, execution | MCP tools; Skills progressive disclosure |
| 5 | Context lifecycle | Compaction, clearing old tool results, offloading large outputs, prompt caching | Anthropic context engineering; provider compaction and cache controls |
| 6 | Memory and retrieval | Notes files, AGENTS.md, RAG, cross-session memory | Anthropic structured note-taking; LangGraph Store |
| 7 | Delegation | Sub-agents with clean contexts; handoffs that transfer control; agents as tools | Anthropic sub-agent architectures; OpenAI Agents SDK handoffs |
| 8 | State and durable execution | Checkpoints per step, threads, resume after failure or long pauses | LangGraph durable execution; 12-Factor (launch/pause/resume, stateless reducer) |
| 9 | Human in the loop | Approvals, questions, edits; suspend and resume across time | 12-Factor (contact humans with tool calls); LangGraph `interrupt`; MCP elicitation |
| 10 | Guardrails | Input, output, and tool checks that can trip and halt | OpenAI Agents SDK guardrails |
| 11 | Permissions, sandboxing, authority | What may run where; isolation; authority labels on content | Authority hierarchy; sandbox runtimes; MCP roots |
| 12 | Lifecycle hooks | Deterministic code at defined points: session, prompt, tool, compaction, sub-agent, stop | Claude Code hooks (about 30 events, 15 can block) |
| 13 | Observability | Traces and spans over model calls, tools, handoffs, guardrails | OpenAI tracing (on by default); OTel GenAI conventions |
| 14 | Errors and retries | Retry policies, compacting errors into context, failure outcomes | 12-Factor (compact errors into context window); LangGraph retry policy |
| 15 | Cost and budgets | Token, time, and money limits; per-agent and shared pools | JAZ BudgetPool; production best practices |
| 16 | Model routing | Choosing models per step; provider differences in tool formats | Orchestration logic in most harnesses |
| 17 | Triggers, streaming, and UI | Start from anywhere; stream partial output; user interruption | 12-Factor (trigger from anywhere); AG-UI |
| 18 | Verification and evaluation | Self-check loops, tests, evals, harness optimization | ETCLOVG Verification; Meta-Harness |

## How existing harnesses expose these concerns

Each system picks a small primitive set, and the sets disagree on what is primitive. That disagreement is the design space the Harness ISA has to settle.

| System | What it treats as primitive | Concerns it makes first-class | Notably absent or implicit |
| --- | --- | --- | --- |
| [OpenAI Agents SDK](https://openai.github.io/openai-agents-python) | Agents, handoffs / agents-as-tools, guardrails, sessions, tracing | 1, 7, 10, 13 | Durable resume; context lifecycle |
| [LangGraph](https://docs.langchain.com/oss/python/langgraph/durable-execution) | Graph nodes, checkpointer, threads, store, `interrupt` | 1, 8, 9 | Hooks; budgets |
| [Claude Code hooks](https://docs.claude.com/en/docs/claude-code/hooks) | Tools, sub-agents, compaction, about 30 hook events grouped by session, tools, turn, sub-agents, context | 4, 5, 7, 12 | Portable API outside Claude Code |
| Pi | Four tools (read, write, edit, bash), extension events such as `before_agent_start`, `tool_call`, `tool_result` | 1, 4, 12 | Most concerns pushed to packages |
| [MCP](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle.md) | Server: tools, resources, prompts. Client: sampling, elicitation, roots. Capability negotiation at init | 4, 9, 11 | The loop itself; state |
| [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) | Principles, not primitives; agent as a stateless reducer over a thread | 1, 2, 3, 8, 9, 14 | Not a runtime |
| Meng et al. survey | Execution loop, tool registry, context manager, state store, lifecycle hooks, evaluation interface | 1, 4, 5, 8, 12, 18 | Governance |

Two patterns stand out. **Hook vocabularies are converging**: Claude Code, Pi, and JAZ all fire events before and after tool calls, at session start, and at stop; Claude Code adds compaction and sub-agent events. And **MCP already standardizes three things the ISA needs**: capability negotiation (`CAPS`), a server asking the client's model for a completion (`sampling`, an `INFER` from the tool side), and a server asking the user a question (`elicitation`, a `HUMAN.ASK`).

## Interoperability standards

Beside the harnesses themselves, a small set of open standards define how harness components talk to the outside world. A harness ISA has to interoperate with them rather than replace them. Each standardizes a *boundary*, not the harness's internals.

| Standard | Boundary it standardizes | Key pieces | Relation to the Harness ISA |
| --- | --- | --- | --- |
| **MCP** (Model Context Protocol) | Agent to tools and data | JSON-RPC client–server; capability negotiation at initialization; server features (tools, resources, prompts); client features (sampling, elicitation, roots) | A transport for `CALL` and namespaces. Its sampling and elicitation are the re-entrant tool channels (a tool asking the model or the user) |
| **A2A** (Agent2Agent; Linux Foundation, absorbed IBM's ACP in August 2025) | Agent to agent, across frameworks and vendors | Agent Card at a well-known URL (identity, skills, endpoint, capabilities, auth); stateful tasks with a lifecycle including input-required; streaming; push notifications to webhooks; JSON-RPC over HTTP, plus gRPC | A transport for `SPAWN` and channels to agents outside the environment. Agent Card ≈ a published `CAPS`; task ≈ receipt; push notification ≈ interrupt |
| **ANP** (Agent Network Protocol) | Open, decentralized networks of peer agents | Decentralized identifiers (W3C DIDs), protocol negotiation | Long-range; relevant only if HB agents join open networks |
| **Agent Skills** (SKILL.md) | Packaged instructions and resources for agents | Folder with a SKILL.md, optional scripts, progressive disclosure | The content format for skills; HB adds manifests and programs (skill binding) |
| **AGENTS.md** | Project instructions for coding agents | A conventional Markdown file at the repository root | Content for a `store` segment in the prime agent's plan |

Layering, as surveys describe it: MCP for tool access at the base, A2A for collaboration between agents, ANP for open discovery. The Harness ISA sits beside these as the *programming interface* inside one environment; these are its *wire formats* to the outside. Details, including how A2A maps onto receipts and interrupts, are in Reference: Networking and Agent Communication.

## Coverage matrix: concerns vs. Harness ISA v0.2

v0.2 fully covers 5 of 18 concerns, partly covers 10, and misses 3. Most partial cases share one cause: v0.2 specifies the instruction but not the **outcome shapes, lifecycle events, or guarantees** around it.

| # | Concern | v0.2 mechanism | Status | Gap |
| --- | --- | --- | --- | --- |
| 1 | Agent loop | `RUN`, `INFER`, `AWAIT` | Covered | — |
| 2 | Context assembly | `CTX.*`, mission slot | Partial | No ordering or stable-prefix notion for caching |
| 3 | Output decoding and validation | Hidden inside `INFER` | **Missing** | No `rejects[]`, no illegal-call trap, no repair policy |
| 4 | Tool registry and dispatch | `TOOL.CALL`, `CAPS` | Partial | No tool registration, schema contract, progressive disclosure, or list-change notice |
| 5 | Context lifecycle | Derived compaction, `Zcompact` | Partial | No tool-result clearing or offload; no cache semantics |
| 6 | Memory and retrieval | `STORE.*` | Covered | — |
| 7 | Delegation | `CTX.FORK`/`NEW` + `RUN` | Covered | Handoff (transfer control and end) is derivable but unnamed |
| 8 | Durable execution | Extension `P` | Partial | LangGraph and 12-Factor treat it as core; v0.2 makes it optional |
| 9 | Human in the loop | `HUMAN.ASK` | Covered | Resume after restart depends on 8 |
| 10 | Guardrails | `EVT.ON` at Enter/Complete | Partial | No standard trip or verdict shape |
| 11 | Permissions and authority | Caps, mission/data split | Partial | No authority levels, privilege modes, or trap routing to policy |
| 12 | Lifecycle hooks | Four phases per operation | Partial | Session, compaction, and sub-agent lifecycle events not named |
| 13 | Observability | Events, cost on handles | Partial | No trace or span identity; no alignment with OTel GenAI |
| 14 | Errors and retries | Trap kinds | Partial | Invisible-retry policy and error-into-context behavior unspecified |
| 15 | Cost and budgets | Budgets, cost records, scoping | Covered | — |
| 16 | Model routing | `model` argument to `INFER` | Partial | No model capability description; tool formats differ by model |
| 17 | Triggers, streaming, UI | `HUMAN.EMIT` | **Missing** | No streaming results, partial output, or user interruption mid-call |
| 18 | Verification and evaluation | None | **Missing** | Possibly out of scope; at least a conformance suite |

## Questions for v0.3

Reading v0.2 from the harness side, and matching each gap to a CPU idea in the companion reference, gives eight questions.

- [ ] **Decode outcomes (concern 3).** Specify `INFER`'s `rejects[]`, an illegal-call trap, and repair policy as invisible, contained, or fatal. CPU parallel: the illegal-instruction exception.
- [ ] **Durability in the base? (8)** 12-Factor's "stateless reducer" suggests every context and loop must be serializable by definition, which would make most of extension `P` a base property rather than an instruction.
- [ ] **Privilege and authority (11).** Map the authority hierarchy onto privilege levels, with traps routed upward for policy decisions. CPU parallel: U/S/M modes, vertical vs. horizontal traps, and code written for one level running under another.
- [ ] **Lifecycle events (12).** Add named session, turn, compaction, and delegation events alongside the per-operation phases, aligned with the vocabulary Claude Code, Pi, and JAZ already share.
- [ ] **Streaming and interruption (17).** Is an `INFER` handle a stream with partial results, and can the user interrupt mid-call? CPU parallel: asynchronous interrupts, as opposed to synchronous exceptions.
- [ ] **Cache hints (2, 5).** Express "keep this prefix stable" as a hint that never changes semantics. CPU parallel: RISC-V's non-temporal locality hint extension.
- [ ] **Concurrency model (6, 7).** What do forks see when they write the same store? CPU parallel: the memory consistency model.
- [ ] **Model encoding layer (16).** Models are post-trained against specific harness tool formats, so the tool-call encoding may need to vary per model while the ISA above it stays fixed. CPU parallel: one ISA, several encodings (compressed instructions).

## Reference documentation

| Document | Use it for |
| --- | --- |
| [Agent Harness, Agentic AI Knowledge Base](https://agentic-ai.readthedocs.io/en/latest/AgentHarness/agent-harness/) | Harness definition, ETCLOVG and H=(E,T,C,S,L,V) summaries, maturity levels, authority hierarchy, links to surveys |
| [12-Factor Agents (HumanLayer)](https://github.com/humanlayer/12-factor-agents) | Production principles: own prompts, context, and control flow; pause/resume; stateless reducer |
| [OpenAI Agents SDK docs](https://openai.github.io/openai-agents-python) | Agents, handoffs, guardrails, sessions, tracing |
| [LangGraph durable execution](https://docs.langchain.com/oss/python/langgraph/durable-execution) and [human-in-the-loop](https://docs.langchain.com/oss/python/langgraph/human-in-the-loop) | Checkpointers, threads, durability modes, `interrupt` |
| [Claude Code hooks reference](https://docs.claude.com/en/docs/claude-code/hooks) and [event list overview](https://agentsroom.dev/blog/claude-code-hook-events-lifecycle) | Hook events, blocking semantics |
| [Hugging Face context course: hook events](https://huggingface.co/learn/context-course/unit5/hook-events) | Cross-harness comparison of hook events, including Pi |
| [MCP specification, lifecycle](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle.md) | Capability negotiation; tools, resources, prompts; sampling, elicitation, roots |
| [Anthropic context engineering, summarized](https://agentic-ai.readthedocs.io/en/latest/ContextEngineering/anthropic/) | Compaction, structured note-taking, sub-agent architectures |
| [Context engineering levers across providers](https://www.developersdigest.tech/blog/context-engineering-for-agents-explained) | Caching, compaction, tool-result clearing API knobs |
| [JAZ](https://arxiv.org/pdf/2609.26891), [Quine](https://arxiv.org/html/2603.18030v1), [Pie](https://arxiv.org/html/2510.24051v1) | Primitive-level designs already used in drafts v0.1 and v0.2 |

Added with the interoperability section: [A2A Protocol Specification v1.0](https://a2a-protocol.org/v1.0.0/specification/) and [Ehtesham et al., A Survey of Agent Interoperability Protocols: MCP, ACP, A2A, ANP](https://arxiv.org/abs/2505.02279v2).
