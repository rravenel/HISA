# Reference: Survey of Computing Principles

Oct 20, 2026 · @Rene Ravenel

A high-level sweep of established computing disciplines, looking for principles the Harness ISA should borrow. The index to the reference library.

## Purpose and the reference library

Language models are new; nearly every problem a harness faces is not. Scheduling, isolation, durability, coordination, trust, and optimization have decades of settled answers. This survey sweeps the major computing disciplines at a high level, records the principles that transfer, and decides which disciplines deserve a deep reference of their own.

| Reference | Role | Status |
| --- | --- | --- |
| Reference: CPU ISA Design | Instruction sets: layering, base plus extensions, traps, privilege, profiles | Exists |
| Reference: Harness Principles | The target: 18 harness concerns, how today's systems handle them | Exists |
| Reference: JIT Compilation Principles | Moving work from the model into code, and cheapening the inference that stays | Exists |
| **Reference: POSIX and Operating Systems** | The rung above the ISA: processes, files, signals, pipes, permissions, the Unix philosophy | New, alongside this survey |
| **Reference: Compiler Principles** | Front, middle, and back ends; intermediate representations; optimization passes | New, alongside this survey |
| Reference: Survey of Computing Principles | This document: the map of everything else | New |

**Added after this survey:** **Reference: Security Principles** (capabilities, the confused deputy, the Rule of Two, CaMeL, information-flow control) and **Reference: Networking and Agent Communication** (local vs. remote, sockets as the channel model, MCP and A2A, backpressure, liveness). Those cover candidates 1 and 2 below, the latter folded into networking.

## The map

"Status" says whether the project already uses the idea, should adopt it, or covers it in a deep reference.

| Domain | Established principles | What it gives the Harness ISA | Status |
| --- | --- | --- | --- |
| Computer architecture | Small frozen base plus extensions; traps; privilege modes; profiles | The ISA framing itself | Deep reference (CPU ISA) |
| Operating systems, POSIX | Processes, file descriptors, signals, pipes, permissions, everything-is-a-file, mechanism vs. policy | The environment interface, agent lifecycle, IPC, namespaces | Deep reference (new) |
| Compilers | Front/middle/back ends, IR, SSA, passes, instruction selection, register allocation, scheduling | Plans as an IR, codecs as back ends, composition as register allocation | Deep reference (new) |
| Runtimes, JIT | Tiering, profiling, guards, deoptimization, partial evaluation | The authoring agent's method | Deep reference (JIT) |
| Programming languages | Algebraic effects and handlers, type and effect systems, continuations, capability-safe languages | Tool calls as effects with handlers; effect classes; serializable continuations for durable waits | Adopt |
| Databases | Write-ahead logging, ACID, MVCC and snapshots, declarative queries with cost-based planners, materialized views, indexes | The journal; composition plans as queries with a cost-based planner; summaries as materialized views | Partly used; adopt planning |
| Distributed systems | Idempotency, at-least-once delivery, sagas, leases, consensus, CAP, logical clocks, actor supervision | Retries, rollback with side effects, peer coordination, ordering across agents | Partly used |
| Networking | End-to-end argument, layering, robustness principle, flow control and backpressure, content negotiation | Where checks belong; backpressure on fan-out; codecs negotiated per model | Adopt |
| Security | Saltzer and Schroeder's principles (least privilege, complete mediation, fail-safe defaults, separation of privilege), capabilities, the confused deputy, information flow | Prompt injection is a confused-deputy problem; scope capabilities rather than filter content | Adopt; candidate deep reference |
| Virtualization | Trap-and-emulate (Popek and Goldberg), nested virtualization | A harness running inside an outer harness that traps and vets its calls | Adopt |
| Software engineering | Semantic versioning, Hyrum's law, strangler fig, golden-master tests, CI/CD | ISA evolution; prototype-to-production path | Partly used |
| Testing and verification | Property-based testing, model checking, conformance suites, differential testing | Conformance for both entry points; checking concurrency semantics | Adopt |
| Observability | Distributed tracing, spans and context propagation | Journal as traces; trace ids across agents | Partly used |
| Scheduling and queueing | Little's law, work-conserving schedulers, fairness, admission control, priority inversion | Fan-out limits on local models; fairness among siblings; who waits on whom | Adopt |
| Information theory | Compression, rate–distortion, channel capacity | Context as a limited channel; summaries as lossy compression with a measurable distortion cost | Lens |
| Control theory | Feedback loops, stability, hysteresis, damping | Agent loops as closed loops; promotion and demotion need hysteresis to avoid oscillation | Adopt |
| Human–computer interaction | Mixed-initiative interaction, direct manipulation, explainability | When to ask the user versus act; approval design | Lens |
| Economics of computing | Caching economics, amortization, metering and quotas | Break-even rules for promotion, cache warming, budgets | Partly used |

## The strongest new imports

Six ideas from the map change how the project should think, not just what it names.

**1. Prompt injection is a confused deputy (security).** Norm Hardy named the problem in 1988: a compiler with legitimate rights to write a billing file was tricked by a user into overwriting it. The deputy wasn't compromised; it was confused about whose authority it was using. An agent reading untrusted text while holding tools is the same structure, and current security writing says so directly. The established fix is capability scoping, not better reading: if the part of the system exposed to attacker-controlled text cannot take consequential action, injection degrades to wasted tokens. *For HB:* this strengthens per-agent namespaces and authority labels, and suggests a standard pattern: a quarantined reader agent with no tools that processes untrusted content and returns only structured data to a privileged agent.

**2. Tool calls are algebraic effects (programming languages).** In languages with effect handlers, code *performs* an effect ("I need X") and a handler, installed by the caller, decides how to fulfill it: real I/O in production, a stub in tests, a replay in recovery. That is exactly a tool call plus an `ON` hook, and recent work applies it to LLM programs directly: one 2025 paper treats LLM calls, I/O, and concurrency as effects discharged by handlers and reports a 10× speedup on a Tree-of-Thoughts case through handler-level parallelism; another, Pangolin, models LLM interactions as first-class algebraic effects. *For HB:* handlers give a precise semantics for hooks, testing with mock handlers, replay from the journal as a handler, and read-only modes as a handler that denies non-read effects.

**3. The end-to-end argument (networking).** Saltzer, Reed, and Clark (1984): some functions can be implemented completely and correctly only with the knowledge of the application at the endpoints; lower layers may implement them partially, but only as an optimization. *For HB:* the ISA should not try to guarantee task-level correctness (that the answer is right, that the customer was helped); only the application can. The ISA provides mechanisms (journal, guards, verdicts) and leaves correctness policy to the endpoints. It's a check against the base growing features that belong to applications.

**4. Composition as query planning (databases).** SQL separates *what* data you want from *how* to fetch it, and a cost-based planner chooses the how. Composition plans can work the same way: a program states what a view must contain (the mission, the relevant prior decisions, the open ticket) and a planner chooses how, comparing costs: full events vs. summaries, cache-friendly ordering, which retrieval index to use. Summaries are **materialized views**, precomputed and refreshed when their sources change. *For HB:* plans become declarative, with the composer as an optimizer rather than hand-written assembly.

**5. Trap-and-emulate (virtualization).** Popek and Goldberg's 1974 conditions for virtualizability rest on privileged operations trapping to a monitor that emulates them. *For HB:* a harness can run unmodified inside an outer harness (an enterprise policy layer, a parent authoring agent, a test rig) if every privileged instruction traps outward. This is the missing layer below the ISA in the CPU reference's stack, and it is how an authoring agent can safely run the target harnesses it builds.

**6. Hysteresis (control theory).** A thermostat that switches at one exact temperature chatters on and off. Systems avoid oscillation with separate on and off thresholds. *For HB:* promotion and demotion of programs, model-role fallbacks, and cache keepalive decisions all need hysteresis, or they will thrash. The JIT reference's "deoptimization storms" are this problem by another name.

## Candidates for further deep references

In priority order. Each earns a deep reference when the project is about to design the area it covers.

1. **Security principles.** Highest priority: capabilities, the confused deputy, information flow, and authority are central to v0.4 (authority labels, guardrails, privilege levels, secrets, promotion gates). Write before specifying those.
2. **Distributed systems.** Before specifying IPC, leases, peer coordination, and crash recovery across agents: idempotency, sagas, consensus, clocks, supervision.
3. **Databases.** Before specifying `QUERY`, store consistency, and declarative composition: transactions, isolation levels, query planning, materialized views.
4. **Programming languages: effects.** Before formalizing hook semantics and sandbox bindings: algebraic effects, handlers, continuations, capability-safe languages.
5. **Testing and verification.** Before building the conformance suite.

The remaining domains (networking, virtualization, observability, scheduling, information theory, control theory, HCI, economics) are adequately covered by their rows and highlights above for now.

## Sources

- [Saltzer & Schroeder, The Protection of Information in Computer Systems (1975), summary](https://en.wikipedia.org/wiki/The_Protection_of_Information_in_Computer_Systems)
- [Prompt injection as a confused deputy (2026)](https://tianpan.co/blog/2026-05-17-prompt-injection-confused-deputy-capability-scoping) and [Cloud Security Alliance research note](https://labs.cloudsecurityalliance.org/research/csa-research-note-ai-agent-confused-deputy-prompt-injection/)
- [Composable Effect Handling for Programming LLM-integrated Scripts (LMPL 2025)](https://conf.researchr.org/details/icfp-splash-2025/lmpl-2025-papers/20/Composable-Effect-Handling-for-Programming-LLM-integrated-Scripts) and [Pangolin: LLMs with algebraic effect handlers (LMPL 2025)](https://conf.researchr.org/details/icfp-splash-2025/lmpl-2025-papers/5/Programming-Large-Language-Models-with-Algebraic-Effect-Handlers-and-the-Selection-Mo)
- [Saltzer, Reed & Clark, End-to-End Arguments in System Design (1984)](https://www.cs.columbia.edu/~nieh/teaching/e6118_s00/papers/endtoend.pdf)
- From general knowledge, not fetched this session: Hardy, "The Confused Deputy" (1988); Popek & Goldberg, virtualization requirements (1974); Little's law; Hyrum's law; Postel's robustness principle.
