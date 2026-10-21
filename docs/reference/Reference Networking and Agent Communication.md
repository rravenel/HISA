# Reference: Networking and Agent Communication

Oct 20, 2026 · @Rene Ravenel

What networking and distributed systems already know about processes talking to each other, applied first to agent-to-agent communication.

## Why networking applies: local versus remote

Early in this project a subagent was described as a function call. Networking's most important warning says to be careful with that framing. In "A Note on Distributed Computing" (Sun Labs, 1994), Jim Waldo and colleagues argued that systems which try to make remote calls look like local ones fail, because remote interaction differs in four ways a local call hides:

- **Latency:** remote operations are orders of magnitude slower.
- **Memory access:** a pointer in one address space means nothing in another.
- **Partial failure:** a remote call can fail, or succeed without the caller learning it did, while the caller keeps running. Local calls don't fail halfway.
- **Concurrency:** remote objects are used by others at the same time.

Their conclusion: interfaces must be designed knowing whether they cross a boundary, rather than papering over it.

Every one of these applies to a subagent, even on the same machine. An inference call takes seconds. A subagent can't follow a reference into its parent's context unless handed a capability. A subagent can crash mid-task after writing files. And peers run concurrently. So a subagent is a **remote** call in Waldo's sense, whether it runs in-process or on another continent. The Harness ISA's receipts, timeouts, journaled results, and explicit capabilities are exactly the interface Waldo argued for; this reference extends that thinking to agent-to-agent communication generally.

## Core networking principles and their agent counterparts

| Principle | In networking | For agent communication |
| --- | --- | --- |
| Layering | Each layer (link, network, transport, application) offers one service to the layer above and hides the one below | Separate the channel abstraction agents use from the transport (in-process queue, local socket, HTTP, a remote protocol) |
| End-to-end argument | Only the endpoints can guarantee correctness; lower layers help only as optimizations | Only the requesting agent can judge whether a delegated task was done right; transport acknowledgments mean delivery, not success |
| Robustness principle | Postel: be conservative in what you send, liberal in what you accept. Later critiqued for letting sloppy implementations spread | Strict output schemas when sending; tolerant decoding with repair when receiving; but log every repair, so sloppiness is visible and doesn't become the norm |
| Flow control and backpressure | Receivers limit how fast senders may send; full buffers push back | Bounded mailboxes; a parent fanning out to many subagents slows down when they or the local model are saturated |
| Congestion control | Senders back off when the network is overloaded (TCP's additive increase, multiplicative decrease) | Back off on provider rate limits; adapt fan-out to observed latency and errors |
| Delivery semantics | At-most-once, at-least-once; exactly-once is achieved by idempotent processing, not by the network | Messages and tool calls carry idempotency keys so retries after a crash don't duplicate effects |
| Timeouts and keepalives | Detect dead peers; hold idle connections open with heartbeats | Detect dead or stuck subagents with heartbeats and deadlines; distinct from cache keepalive |
| Naming, addressing, discovery | DNS maps names to addresses; service discovery finds providers | Agents and channels have names in a namespace; directories (or Agent Cards) let agents find peers |
| Content negotiation | Client and server agree on formats (HTTP `Accept`) | Agents agree on message schemas and modalities; codecs negotiate per model |
| Fallacies of distributed computing | Peter Deutsch's list, extended by James Gosling: the network is reliable, latency is zero, bandwidth is infinite, the network is secure, topology doesn't change, there is one administrator, transport cost is zero, the network is homogeneous. All false | All false for agents too: providers fail, inference is slow, context is finite, peers can be compromised, agents come and go, several parties set policy, tokens cost money, models differ |

## Communication patterns as channel types

**Sockets are the unifying abstraction.** The Berkeley sockets API (1983) gave Unix one interface for every kind of communication: a socket is a file descriptor, created with a type that fixes its semantics (stream or datagram), bound to an address, and either listening for connections or connecting to one. The same calls work for a process on the same machine (Unix domain sockets) and across the internet. That's the property agents need: one channel interface, with the transport chosen separately.

The patterns agents need map onto established ones:

| Pattern | Established form | Semantics | Agent use |
| --- | --- | --- | --- |
| Request/response | RPC; HTTP | One request, one reply, with a deadline | Subagent as a task: delegate and receive a result (the default `SPAWN` and `AWAIT`) |
| Stream | TCP socket; pipe | Ordered, reliable byte or message stream between two endpoints | Pipelines where one agent's output feeds another's input; streaming partial results |
| Datagram or message | UDP; message queues | Independent messages, possibly unordered | One-off notifications and questions between peers |
| Mailbox | Actor mailboxes (Erlang); message queues | Messages queue for a named recipient that processes them in order | Each agent's inbox; interrupts are mailbox deliveries at instruction boundaries |
| Publish/subscribe | MQTT topics; event buses | Senders publish to a topic; any number of subscribers receive | Broadcasts to a team ("schema changed"); progress feeds a supervisor watches |
| Shared state with locks | Files plus `flock`; leases | Coordination through a common resource rather than messages | The dev-team case: who owns `auth.py` right now |
| Listen and accept | Server sockets | A named endpoint that accepts connections from unknown peers | An agent offering a service to others it didn't spawn |

Two decisions shape all of these. **Who may connect:** a listening endpoint is reachable by anyone who can name it, so in a capability system the name itself should be a capability handed out deliberately. **Who owns the conversation's history:** in HB every message is a journal event, so all of these patterns are durable and replayable regardless of transport.

## Existing agent protocols and how they layer

Three protocols matter as of late 2026; a fourth has merged away.

- **MCP (Model Context Protocol)** connects an agent to tools and data: client–server, JSON-RPC, capability negotiation at initialization, with tools, resources, and prompts on the server side and sampling, elicitation, and roots on the client side. It is the agent-to-tool layer.
- **A2A (Agent2Agent)**, started by Google and donated to the Linux Foundation, connects agents to other agents, including ones built on different frameworks and opaque to each other. Its pieces map closely onto HB concepts:
  - an **Agent Card**, a JSON document at a well-known URL describing an agent's identity, skills, endpoint, capabilities (streaming, push notifications, extensions), and authentication: discovery plus a published `CAPS`;
  - **tasks** with a defined lifecycle, including an input-required state where the agent pauses: receipts and durable waits;
  - **streaming** of status and partial artifacts, and **push notifications** to a client webhook for long-running or disconnected work: interrupts delivered over the network;
  - JSON-RPC over HTTP, with gRPC added later.
- **IBM's Agent Communication Protocol (ACP)**, a REST-native messaging protocol launched in March 2025, was merged into A2A in August 2025 under the Linux Foundation.
- **ANP (Agent Network Protocol)** targets open, internet-scale networks of peer agents, with decentralized identifiers (W3C DIDs) for trustless identity and dynamic protocol negotiation.

Surveys describe them as layers: MCP for tool access at the base, A2A (with ACP folded in) for collaboration between agents, and ANP for open, decentralized discovery.

*Harness reading:* these are **transports and wire formats**, not ISAs. They define how bytes move between agents built by different parties. HB defines what a program can do. The two fit together the way a sockets API fits over TCP: HB channels and `SPAWN` can have an A2A transport for agents outside the local environment, and an MCP transport for tools.

## Implications for the Harness ISA

1. **One channel abstraction, many transports (R51).** Define channels once, like sockets: a channel has a type (request/response, stream, message, mailbox, pub/sub) and a name that is a capability. Transports (in-process queue, local socket, HTTP, A2A) are chosen separately and invisible to programs.
2. **Design every agent interface as remote.** Per Waldo: all inter-agent calls have deadlines, can fail partially, return receipts, and never share memory except through granted capabilities. This is already the shape of v0.3; make it a stated rule so no future "fast local path" quietly drops it.
3. **Remote agents via A2A (R52).** `SPAWN` can target an agent outside the environment through an A2A transport: its Agent Card is read as that agent's `CAPS`, its task becomes the receipt, its push notifications become interrupts. And an HB agent can be *exposed* as an A2A server, publishing an Agent Card generated from its namespace and skills.
4. **Backpressure (R53).** Mailboxes are bounded; sends to a full mailbox block, fail, or shed load by declared policy. Fan-out adapts to observed latency and rate limits instead of launching everything at once.
5. **Liveness (R54).** Long-running agents emit heartbeats; a parent sets deadlines; a missed heartbeat is an event the parent or supervisor handles (retry, reassign, give up). Distinct from cache keepalive.
6. **Idempotent messaging.** Every message and call carries an idempotency key, so delivery can be at-least-once with effectively-once processing.
7. **End-to-end acknowledgment.** A delegated task isn't done when the transport says delivered, or even when the subagent says done; it's done when the requester's own check passes. That is where verification belongs.
8. **Peers are untrusted by default.** Messages from other agents carry their sender's integrity label (see the security reference). In multi-agent systems, peer channels are the propagation path for injected instructions.

The broader point: the project has been converging on an actor-style, capability-secured messaging system with durable logs. That is a well-studied design (Erlang/OTP, E, Akka, Kafka-style logs), so most of its pitfalls are known in advance.

## Sources

- [Waldo, Wyant, Wollrath & Kendall, A Note on Distributed Computing (1994)](https://waldo.scholars.harvard.edu/publications/note-distributed-computing)
- [A2A Protocol Specification v1.0](https://a2a-protocol.org/v1.0.0/specification/) and [A2A summary](https://a2a-protocol.org/llms.txt)
- [Ehtesham et al., A Survey of Agent Interoperability Protocols: MCP, ACP, A2A, ANP](https://arxiv.org/abs/2505.02279v2)
- [AI agent protocols explained, including the ACP–A2A merger](https://www.virtua.cloud/learn/en/concepts/ai-agent-protocols-explained)
- [MCP specification: lifecycle and capabilities](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle.md)
- [Saltzer, Reed & Clark, End-to-End Arguments in System Design](https://www.cs.columbia.edu/~nieh/teaching/e6118_s00/papers/endtoend.pdf)
- From general knowledge, not fetched this session: Berkeley sockets API (4.2BSD, 1983); Postel's robustness principle (RFC 761/793) and its later critiques; Deutsch's fallacies of distributed computing; TCP congestion control.
