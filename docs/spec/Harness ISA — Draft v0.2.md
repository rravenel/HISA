# Harness ISA — Draft v0.2

Oct 19, 2026 · @Rene Ravenel

v0.2 shrinks the base from 20 instructions to 18, moves policy-heavy operations into named extensions, and adds the environment contract that v0.1 lacked. Each change traces to a source in the next two sections.

## What changed since v0.1

| Change | v0.1 | v0.2 | Why |
| --- | --- | --- | --- |
| Environment contract | Implicit | **Harness Environment Interface (HEI)** defines initial state, trap handling, preemption, budgets | RISC-V treats the ISA as one part of an execution environment interface |
| Trap model | Tool call = trap | Four trap kinds: requested, contained, invisible, fatal | RISC-V trap taxonomy |
| `DECODE` | Separate instruction | Folded into `INFER`'s result | "Including too much"; every engine fuses it |
| `CTX.COMPACT`, `CTX.SNAPSHOT` | Base | Compaction derived; snapshot moved to extension `P` | Lossy, policy-laden ops don't belong in a frozen base |
| `CTX.READ` | Missing | Added | "Leaving out too much": JAZ shows history must be a readable value |
| Mission slot | System prompt | Immutable `mission` on every context, separate from data | Quine's argv/stdin split |
| Agent identity | Open question | `RUN` returns a handle that can be awaited and signaled | Quine (PID), RISC-V harts |
| Events | `pre_*` / `post_*` | Enter, Send, Complete, Exit per operation, with composable effects | JAZ hook model |
| Extensions | Ad hoc `X.*` groups | Named standard letters, `Z` small extensions, `X` vendor space, plus profiles | RISC-V naming, Pie traits |
| Context length | Unstated | Queryable parameter `CTXLEN`, never a constant | Address size is the hardest ISA mistake to undo |

## Lessons from CPU ISA design

RISC-V is the best guide here because its spec explains its own choices in commentary. Six lessons carry over directly.

1. **Small frozen base, layered extensions.** RISC-V defines a minimal base that every implementation must support, plus optional extensions, and keeps the base and each extension constant rather than versioning the whole ISA. Its authors believe it is rare for a new instruction to benefit all applications rather than one domain. *For harnesses:* the base must stay small enough to freeze; new capabilities arrive as extensions.
2. **The ISA is one part of an environment contract.** RISC-V's execution environment interface (EEI) defines initial state, the harts available, memory regions, instruction behavior, and how traps and environment calls are handled; the Linux ABI and RISC-V SBI are examples. *For harnesses:* separate the instruction set from the Harness Environment Interface that hosts it.
3. **Four kinds of trap.** A trap can be contained (handled inside the program), requested (a system call), invisible (handled transparently, e.g. emulating a missing instruction), or fatal. *For harnesses:* a tool call is a requested trap, a hook that handles a call is contained, automatic retry or compaction is invisible, and budget exhaustion is fatal.
4. **The environment owns forward progress.** Software inside the environment is not responsible for making its harts progress, and an environment with more guest harts than host harts must preempt rather than wait for a voluntary yield. *For harnesses:* an agent loop must be preemptible by the harness; cancellation cannot depend on the model cooperating.
5. **Three classic mistakes.** Asanović and Patterson list leaving out too much (Alpha shipped without byte loads), including too much (SPARC register windows), and letting current microarchitecture shape the ISA (MIPS delayed branches). *For harnesses:* don't bake today's model quirks into the ISA, such as vendor tool-call formats, thinking-block syntax, or a fixed context size.
6. **Address size is the hardest mistake to undo.** RISC-V planned 32-, 64-, and 128-bit variants because address size is the one ISA mistake hardest to recover from. *For harnesses:* context length is the address size. It must be a queryable parameter (`CTXLEN`), never an assumption.

Two smaller lessons also apply. RISC-V separates **standard, reserved, and custom** encoding space so vendor extensions never collide with future standard ones. And it names stable bundles, such as "G" for the general-purpose set, so toolchains have one target. Both shape the extension scheme below. Scale is also instructive: the RISC-V base has fewer than 50 instructions and a 2-page reference card, against 3,600 pages for x86.

## Lessons from prior harness work

The published efforts sit at different layers. None defines a portable, frozen base with an environment contract, which is the gap this draft targets.

| Work | Layer | Its primitives | What v0.2 takes | What v0.2 leaves |
| --- | --- | --- | --- | --- |
| [Pie](https://arxiv.org/html/2510.24051v1) (SOSP 2025) | Inference engine | 42 APIs (18 for model execution): embed, forward, sample, explicit KV-page alloc/free/export, command queues, message passing | **Traits** a model advertises and code queries at runtime; explicit resource handles; queues that make dependencies visible to a scheduler | Token- and page-level ops stay out of the base; they become extensions `K` and `T` |
| [SGLang](https://arxiv.org/html/2312.07104v2) | Prompt program DSL | `extend`, `gen`, `select`, `fork`, `join` | Fork as a first-class, prefix-sharing operation | `select` is a constraint, not an instruction |
| [JAZ](https://arxiv.org/pdf/2609.26891) (Sep 2026) | Agent loop | One primitive, `invoke`, plus hooks | History as a readable value passed by reference; hooks at Enter/Send/Complete/Exit emitting composable effects; scoped versus local budgets | The claim that one primitive suffices. True for expressivity, but a portable ISA also needs resource and event semantics that `invoke` leaves to the host |
| [Quine](https://arxiv.org/html/2603.18030v1) (Mar 2026) | OS process model | `sh`, `fork`, `exec`, `exit`, with argv/stdin/stdout/stderr/exit-status channels | Immutable mission separated from data; renewal that keeps identity and mission; three state tiers | Raw POSIX as the substrate. Its own stated limits (no task-relative world, no rollback that keeps lessons) become open questions |
| [AIOS](https://www.themoonlight.io/review/aios-llm-agent-operating-system) (2024) | User-space agent OS | Syscalls via one scheduler; context, memory, storage, and tool managers | Interrupt, snapshot, restore of an in-flight inference (extension `P`) | Fixed eviction policy in the kernel; policy belongs to programs |
| Credo, STITCH (2026, abstracts only) | Harness library and compiler | Typed, reusable harness parts with scope and composition contracts | Confirmation that a typed layer above the ISA is useful | Their parts are standard-library material, not instructions |

Two threads run through all of them. Exposing structure (forks, shared prefixes, dependencies) is what lets a runtime optimize. And each system that went fine-grained paid a measurable price: Pie's unfused loop cost about 1.5 ms per token. v0.2 keeps the base orthogonal and lets implementations fuse common sequences, the same bargain RISC hardware makes with macro-op fusion.

## Design principles

v0.1's six principles stand; v0.2 sharpens three and adds four.

1. **One instruction set, two entry points.** Unchanged: the model reaches it by trapping, programs by library call.
2. **Small frozen base, everything else derived or extended.** Sharpened: if an operation can be built from base instructions without losing correctness, it is derived, not added. Compaction is the test case.
3. **Portable baseline, discoverable extensions.** Sharpened: extensions are named, advertised through `CAPS`, and probed at runtime, as Pie does with traits.
4. **Capabilities, not trust.** Unchanged, plus scoping: a capability or budget is either local to one call or inherited by every child.
5. **Everything observable.** Sharpened: four events per operation (Enter, Send, Complete, Exit), and handlers emit effects that compose rather than mutate state directly.
6. **Explicit cost.** Unchanged: every handle reports what it consumed.
7. **New: no microarchitecture in the ISA.** No vendor message formats, reasoning-block syntax, or tokenizer details in base semantics.
8. **New: context length is a parameter.** Programs read `CTXLEN`; nothing assumes a size.
9. **New: instructions and data travel in separate channels.** A context's mission is set at creation and immutable; everything appended later is data. This is a structural defense against prompt injection, not a complete one.
10. **New: the environment owns progress.** Any running loop can be preempted, signaled, or killed by the harness without the model's cooperation.

## Machine model

The ISA now sits inside a **Harness Environment Interface (HEI)**, which defines everything the instructions assume: initial state, which models and tools exist, how traps are handled, budgets, and preemption.

&#91;embedded content: machine model v0.2 · two callers, six groups, four resources, inside the HEI\]

A **context** is the unit of state: an immutable mission, appended data, a capability set, and a budget. A **running loop** (the `RUN` handle) is the unit of execution, the harness's version of a hart: the environment, not the model, guarantees its progress and can preempt it. State comes in three tiers, following Quine: the context (ephemeral), scoped capabilities and variables inherited by children, and stores (persistent).

Every transfer of control to the harness is one of four trap kinds:

| Trap kind | Harness example | Program sees it? | Execution continues? |
| --- | --- | --- | --- |
| Requested | Model emits a tool call; program calls `TOOL.CALL` | Yes | Yes, with the result |
| Contained | An `EVT.ON` handler answers the call itself | Yes | Yes |
| Invisible | Automatic retry on rate limit; engine-native compaction | No | Yes |
| Fatal | Budget exhausted; policy violation; `SIGNAL` kill | Only as an outcome | No |

## Base instruction set (HB)

The base, named **HB**, has 18 instructions in six groups. Every instruction that can take time returns a **handle**; handles are awaited, signaled, and report cost. Handles to contexts are passed by reference, never copied, which is JAZ's key finding.

| Group | Instruction | Signature | Semantics |
| --- | --- | --- | --- |
| Context | `CTX.NEW` | `(mission, caps, budget) -> ctx` | Create a context with an immutable mission, a capability set, and a budget. |
| Context | `CTX.APPEND` | `(ctx, role, content) -> ctx` | Add data: user input, tool result, attachment. Never alters the mission. |
| Context | `CTX.READ` | `(ctx, range?) -> entries` | Read history as values. New in v0.2. |
| Context | `CTX.FORK` | `(ctx) -> ctx'` | Copy-on-write branch sharing the prefix. |
| Context | `CTX.SLICE` | `(ctx, range) -> ctx'` | Select or drop entries, e.g. a failed branch. |
| Execute | `INFER` | `(ctx, model, constraints) -> h` | One model call. Result: `{text, calls[], stop, cost}`. Constraints: schema, stop, max tokens, tool set. |
| Execute | `RUN` | `(ctx, loop_policy) -> h` | Start a harness-managed agent loop on a context. The handle is the agent's identity, like a PID or hart. |
| Execute | `AWAIT` | `(h[]) -> results[]` | Join any handles: inference, tool, agent, or human. |
| Execute | `SIGNAL` | `(h, sig)` | Cancel, pause, resume, or warn. Delivered by the harness, not the model. |
| Tool | `TOOL.CALL` | `(tool, args) -> h` | Invoke an external function under caps. |
| Store | `STORE.GET` | `(store, key) -> value` | Read persistent state. |
| Store | `STORE.PUT` | `(store, key, value)` | Write persistent state. |
| Store | `STORE.QUERY` | `(store, query, k) -> hits[]` | Keyword, vector, or hybrid search. |
| Human | `HUMAN.EMIT` | `(content)` | Show content to the user. |
| Human | `HUMAN.ASK` | `(prompt, options?) -> h` | Request input or approval. |
| Event | `EVT.ON` | `(op, phase, handler) -> sub` | Subscribe at Enter, Send, Complete, or Exit of any operation. |
| Event | `EVT.OFF` | `(sub)` | Unsubscribe. |
| Meta | `CAPS` | `() -> caps` | Extensions present, tools, models, `CTXLEN`, budget remaining. |

**Event phases** follow JAZ. At Enter a handler may rewrite inputs; at Send it may replace the operation with a supplied result; at Complete it may rewrite the output; any phase but Exit may abort. Effects from multiple handlers are collected and composed, and conflicting writes are refused rather than resolved by order.

**Derived, not base.** Compaction is `CTX.NEW` with the same mission, plus an `INFER` that summarizes, plus `CTX.APPEND` of the summary: Quine's `exec`. Sub-agents are `CTX.FORK` or `CTX.NEW` with narrower caps, then `RUN`. RAG is `STORE.QUERY` then `CTX.APPEND`. Approval gates are `EVT.ON(TOOL.CALL, Enter)` handlers that call `HUMAN.ASK`.

## Extensions and profiles

Extensions follow RISC-V naming. Single capital letters are standard extensions. `Z`-prefixed names are small standard extensions. `X`-prefixed names are vendor space, guaranteed never to collide with a future standard. An implementation's capability string reads like `HB_C_P_Zadapt_Xacme`.

| Name | Adds | Requires | Fallback on plain HB |
| --- | --- | --- | --- |
| `C` Constrained output | `INFER` with `grammar` or `regex` | — | JSON schema, validate and retry |
| `P` Persistence | `CTX.SNAPSHOT`, `CTX.RESTORE`, `RUN.SUSPEND` | — | Rebuild context from `STORE` |
| `S` Speculative dispatch | Start tools or child inferences while the parent streams | — | Sequential infer, then dispatch |
| `K` KV control | `KV.PIN`, `KV.EVICT`, `KV.EXPORT`, `KV.IMPORT` | engine access | Implicit prefix caching |
| `T` Token level | `FORWARD`, `SAMPLE`, `DIST` (top-k distribution) | `K` | Whole-call `INFER` only |
| `E` Sandboxed execution | `EXEC` with resource limits | — | A shell tool under caps |
| `Zcompact` | Engine-native compaction | — | Derived compaction (base) |
| `Zadapt` | Per-call adapter or steering vector | `T` | Prompt conditioning |

**Profiles** give programs a stable target, as RISC-V's "G" does:

- **HG (general):** HB + `C` + `P`. Implementable on any hosted model API today. Skills should target HG.
- **HE (engine):** HG + `S` + `K` + `T`. Requires running beside the model, as Pie does.

The rule for promotion: an extension enters a profile only when it is implementable across vendors and at least two independent harnesses ship it.

## Skill binding

A skill package keeps its prompt and adds two optional parts. A **manifest** names the profile it targets (for example `HG`), any extra extensions it can use, and the capabilities and budget it requests. **Programs** are entry points written against the ISA and run sandboxed under granted capabilities; the model invokes one by name, or an event triggers it. Pie's choice of WebAssembly for the same job is a reasonable default for the sandbox.

The v0.1 code-review example still holds, now targeting HG:

```
def review(ctx, files):
    subs = [CTX.APPEND(CTX.FORK(ctx), "user", REVIEW_PROMPT + f) for f in files]
    outs = AWAIT([INFER(s, "fast", {"schema": Finding}) for s in subs])
    return merge(outs)
```

## Backend concerns

With a fixed ISA, a harness can optimize like a compiler backend. The mapping from v0.1 stands: instruction selection chooses how each step runs, register allocation becomes context allocation, and scheduling orchestrates calls.

v0.2 adds two points from the sources. **Expose dependencies:** because every slow instruction returns a handle, a harness can see which calls are independent and batch them, the role Pie's command queues play. **Fuse, don't widen:** when an orthogonal sequence is slow (Pie measured about 1.5 ms per token for an unfused loop), the implementation fuses it internally rather than adding a compound instruction to the base.

## Open questions for v0.3

Resolved since v0.1: whether `DECODE` is separate (no), where sub-agent identity lives (the `RUN` handle), and what compaction guarantees (it is derived, so the program decides).

- [ ] **Is `RUN` base or derived?** It exists so the environment has a preemptible unit. A program-written loop over `INFER` is also preemptible, which may make `RUN` redundant.
- [ ] **Does HG really fit every hosted API?** Constrained output varies widely; the validate-and-retry fallback may need cost bounds.
- [ ] **Rollback that keeps lessons.** Quine's named gap: undo side effects while preserving what was learned. Candidate extension, possibly `P` plus a transactional `TOOL.CALL`.
- [ ] **Task-relative worlds.** Quine's other gap: capabilities say what an agent may touch, not what its world contains. Is that a capability concern or a new concept?
- [ ] **Model heterogeneity inside `INFER`.** One contract for models with different tool formats and reasoning modes, or selection pushed to callers?
- [ ] **Formal semantics.** RISC-V has a formal memory model. What is the equivalent for concurrent forks writing to one store?
- [ ] **Governance.** Who freezes HB? RISC-V needed a foundation; an ISA without a steward is a proposal.
- [ ] **Reference implementation.** Prototype HG over a hosted API on a minimal harness first, then HE on Pie.

## Sources

- [RISC-V Unprivileged ISA, Introduction](https://docs.riscv.org/reference/isa/v20240411/unpriv/intro.html)
- [RISC-V ISA Extension Naming Conventions](https://docs.riscv.org/reference/isa/unpriv/naming.html)
- [Asanović & Patterson, Instruction Sets Should Be Free: The Case for RISC-V (UCB/EECS-2014-146)](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2014/EECS-2014-146.pdf)
- [Patterson, Instruction Sets Want to Be Free (talk abstract)](https://www.cs.washington.edu/news-events/event-details/?id=2762)
- [Gim et al., Pie: A Programmable Serving System for Emerging LLM Applications (SOSP 2025)](https://arxiv.org/html/2510.24051v1)
- [Zheng et al., SGLang](https://arxiv.org/html/2312.07104v2)
- [Li et al., Harness as a Language (JAZ)](https://arxiv.org/pdf/2609.26891)
- [Ke, Quine: Realizing LLM Agents as Native POSIX Processes](https://arxiv.org/html/2603.18030v1)
- [AIOS review](https://www.themoonlight.io/review/aios-llm-agent-operating-system)
- [Credo](https://arxiv.org/pdf/2608.27790) and [STITCH](https://arxiv.org/pdf/2609.38912), abstracts only; full text was not machine-readable
