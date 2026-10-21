# Harness ISA — Workbook

Oct 19, 2026 · @Rene Ravenel

Scratch pad, brainstorm, and to-do list for the Harness ISA drafts. Loose by design; the drafts are the clean record.

## Scope for v0.2

v0.2 grounds v0.1 in sources and tightens the instruction set; it stops there.

- **In:** design lessons from real CPU ISAs (RISC-V above all, since its spec explains its choices), lessons from published harness-primitive work, a revised base set, an extension and profile scheme.
- **Out:** formal semantics, wire formats, a reference implementation, benchmarks.
- **Stop rule:** about 10–12 sources read in full. When a new source stops changing the instruction table, stop reading and write. Unanswered questions go to Parked, not into more research.

## Decisions after v0.2

Recorded after the CPU and harness reference docs and the review conversation.

- **It's an API that decomposes to the bare minimum.** ISA and POSIX are both useful metaphors; the binding idea is minimal decomposition. Test for every candidate instruction: can it be built from the others without losing correctness or costing more tokens? If yes, it is not base.
- **Tokens are the performance metric.** CPU-side speed of harness code is irrelevant; inference dominates wall-clock time. Wall-clock time is served by parallelism. Any compiler-like optimization optimizes tokens: cache reuse, context allocation, what never enters the window.
- **The intended programmer is an agent.** Target use: one agent that builds or adapts application-specific agents (customer service, invoice reconciliation, and so on) out of these primitives. The instruction set must fit comfortably in a model's context, programs must be short and diffable, and capabilities and budgets are hard limits because generated programs will sometimes be wrong. Related: the self-improving agent experiment could target HB as its substrate.
- **First executable test.** Minimal "silicon" in plain Python implementing five v0.2 instructions (`CTX.NEW`, `CTX.APPEND`, `INFER`, `TOOL.CALL`, `AWAIT`) against a scripted mock model, with an optional real-provider backend run locally. Then a program, an agent loop, using only those five. No existing agent framework underneath. Measure tokens per program and whether independent calls overlap.
- **Both tests run against v0.2 as written.** Where v0.2 is silent, the implementation makes the smallest provisional choice, labels it "not in v0.2," and logs it here as a v0.3 candidate.
- **Venue.** Executable work moves to Claude Code.

## Architecture sketch for v0.3

The loops session reframed the design around seven ideas. v0.3 should be written from these, with the port-test requirements below as its checklist. Each idea notes the established practice it borrows from.

1. **The environment owns the loop.** One event loop schedules contexts the way an OS schedules threads or a CPU advances harts. *Agent mode*: the model drives a context as its processor. *Workflow mode*: a program calls `INFER` as a coprocessor and may emulate the loop itself. Agents are created by an environment call (`SPAWN`), not by an instruction. *(CPU instruction cycle, RISC-V harts, OS scheduler)*
2. **Everything is a tool call, asynchronous by default.** Including the user: a final answer is a call to a user tool, and the user's reply is its result, like a REPL waiting on input. Every call returns a receipt at once; blocking is explicit (`AWAIT`). Plain model text with no call is an illegal instruction (or, as a decode-layer allowance, an implicit user-tool call). The environment coalesces fast completions before reinvoking a model, so async doesn't multiply inference calls. *(12-Factor Agents, REPLs, shell job control, interrupt coalescing)*
3. **Interrupts arrive at instruction boundaries.** The inference call is the atomic instruction. Events for a busy context queue until its call finishes and are injected before the next one. Cancelling an in-flight call is the exception and forfeits its tokens. *(precise interrupts)*
4. **The journal is the truth; contexts are composed views.** Every event is written to a durable local journal before it takes effect. A context is a projection, composed for one inference call from the journal and stores. Compaction is choosing a smaller view, so nothing is lost; handoff is composing a different view, so it stops being a concept; models and agents become swappable. Append-everything is the trivial view, so today's default remains possible. *(event sourcing, write-ahead logs)*
5. **Composition is cache-aware register allocation, measured in tokens.** Calling code decides what enters each context, the way a compiler allocates registers. The API supplies structural edits, per-entry token costs, cache state, spill and reload by reference, and liveness hints. All agents share a canonical prefix so they share cache; cache warmth and keepalive are scheduling inputs. *(compiler register allocation, virtual memory, shared libraries)*
6. **Agents communicate through capability-gated IPC.** The default subagent is an isolated function call. Parents grant channels: RPC to the parent, asks to the user (direct or routed up), spawning, named peer channels, and shared state with leases. Every message is a journal event. *(POSIX pipes, RPC, named pipes, file locks, Capsicum file-descriptor capabilities, RISC-V vertical traps)*
7. **Everything is addressable in a per-agent namespace.** Tools, agents, channels, journals, stores, and contexts have names and a small set of generic verbs, so cross-cutting programs (journaling, permissions, tracing, replay) work on all of them. What an agent can name is its capability set. *(Unix everything-is-a-file, Plan 9 per-process namespaces)*

Unchanged from v0.2: two entry points (the model by tool call, deterministic code by library call), capabilities and budgets as hard limits, tokens as the performance metric, and minimal decomposition as the test for every primitive.

## Prior-art smorgasbord

The design draws on whatever established practice fits, not just ISAs and POSIX. The first group is already in the architecture sketch; the second is new and worth weighing for v0.3. Drawn from general knowledge, not fresh research; each "new" row is a lead to check, not a settled borrowing.

**Already in use:** ISA design and RISC-V (base plus extensions, profiles, traps, harts), POSIX processes and signals, shells and job control, pipes and RPC, compilers and register allocation, event sourcing and write-ahead logs, event loops, precise interrupts and interrupt coalescing, capabilities, Plan 9 namespaces, shared libraries.

**New candidates:**

| Practice | Core idea | Harness application | Strength of fit |
| --- | --- | --- | --- |
| Actor model, Erlang/OTP | Isolated actors with mailboxes; supervisors restart failed children; "let it crash" | Subagents as actors; supervision trees and restart strategies for failed or stuck subagents | Strong |
| Sagas and compensating transactions | Long workflows undo completed steps with explicit compensations | Rollback with side effects (Quine's gap): each side-effecting tool declares its compensation | Strong |
| Idempotency keys, leases, at-least-once delivery | Safe retries and exclusive ownership in distributed systems | Retrying tool calls after a crash; peer coordination over files (the dev-team case) | Strong |
| Working-set model, virtual memory paging | Keep the pages a process is actively using; thrashing when the set exceeds memory | Context composition as working-set selection; a "thrashing" signal when an agent keeps re-retrieving the same material | Strong |
| JIT and tracing compilers | Detect hot paths and compile them to fast code | Repeated agent behavior hardened into deterministic programs, matching "use the model only where code can't"; a natural job for the self-adapting agent | Strong |
| Dynamic linking and lazy binding | Load symbols only when first used | Progressive disclosure of tools and skills: load a definition on first call | Strong, already informal practice |
| Build systems (Make, Bazel), memoization | Recompute only what changed; content-addressed caches | Recompose only changed parts of a context; content-addressed tool results and summaries | Medium |
| Version control (git) | Branches, merges, immutable history | Context forks as branches; merging results of parallel subagents; the journal as an immutable log | Medium |
| Blackboard architecture (classic AI) | Specialists read and write a shared board | Shared store for loosely coupled agents (JAZ hooks also use a blackboard) | Medium |
| Garbage collection, reference counting | Reclaim what nothing references | Retire context entries, channels, and receipts no live agent references | Medium |
| Microkernels | Tiny privileged core; services run as ordinary processes | Minimal base ISA; memory, retrieval, and policy as ordinary services over IPC | Medium |
| Taint tracking, information flow control | Label data by origin and track where it flows | Authority labels on content (R4) that propagate into summaries and composed views | Medium |
| Quotas and fair-share scheduling (cgroups) | Bounded resources per group, fair allocation | Budgets per agent subtree; fairness among sibling subagents | Medium, partly in v0.2 |
| Design by contract, typed interfaces | Pre- and postconditions on calls | Tool schemas and composition contracts (as in STITCH); decode validation | Medium |

## To-do

- [x] Read RISC-V spec rationale: base + extensions, what was left out and why
- [x] Read RISC philosophy sources: covered via Asanović & Patterson only; Hennessy & Patterson's quantitative-design material not read
- [x] Skim x86 vs ARM lessons: covered only by the CISC-translation and manual-size points in Asanović & Patterson
- [x] Re-read Pie paper: which handlers it exposes
- [x] Re-read SGLang primitives
- [x] Read JAZ, Credo, STITCH, Quine for their primitive sets (Credo and STITCH abstracts only)
- [x] Check AIOS syscall list
- [x] Draft v0.2 sections
- [ ] Review pass on v0.2 instruction table, ideally with fresh eyes
- [x] Move unresolved questions to Parked
- [ ] Next: sketch an HB conformance test list (see Brainstorm)

Added after the loops session:

- [x] Written port test against v0.2
- [x] Integrate loops-session raw notes (architecture sketch, prior-art smorgasbord, R1–R16)
- [ ] Decide whether the executable test still targets v0.2's five instructions, or a v0.3 core (journal + `COMPOSE` + async calls)
- [ ] Draft v0.3 from the architecture sketch, using R1–R16 as its checklist
- [ ] Check the strongest smorgasbord leads (actors/OTP supervision, sagas, working-set model, JIT hardening) against sources

* [x] Integrate post-v0.3 raw notes (R17–R28, cases V–X)
* [ ] After the first build: draft v0.4 from R17–R28 plus build findings; re-run cases A–X

## Research log

| Source | Takeaway for the Harness ISA |
| --- | --- |
| [RISC-V unprivileged spec, intro](https://docs.riscv.org/reference/isa/v20240411/unpriv/intro.html) | ISA is one part of an **execution environment interface** (EEI). Base is minimal and frozen; new instructions arrive as optional extensions, never as a new base version. Traps come in four kinds: contained, requested, invisible, fatal. The environment, not the program, guarantees forward progress and may preempt. |
| [Asanović & Patterson, "Instruction Sets Should Be Free"](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2014/EECS-2014-146.pdf) | Three classic ISA mistakes: leaving out too much, including too much, and letting today's microarchitecture leak into the ISA (delay slots). Base-plus-extension needs a small dependable core, standard optional extensions, and reserved space for custom ones. Address size is the one mistake that is hardest to undo. |
| [RISC-V naming conventions](https://docs.riscv.org/reference/isa/unpriv/naming.html) | Named bundles ("G" = IMAFD + Zicsr + Zifencei) give software a stable target. Non-standard extensions get their own namespace (X prefix). |
| [Pie (SOSP 2025)](https://arxiv.org/html/2510.24051v1) | 42 APIs, 18 for model execution, grouped into **traits** a model advertises and code queries at runtime. Explicit alloc/free of KV pages and embeddings; command queues make dependencies explicit for batching. Fine-grained ops cost about 1.5 ms/token vs. a fused loop; authors accept it for composability. |
| [JAZ, "Harness as a Language"](https://arxiv.org/pdf/2609.26891) | One primitive, `invoke`, plus hooks. Sub-agents are recursive invokes; the agent's own history is a variable it can pass by reference. Matches or beats specialized memory (Letta) and self-improvement (ACE) harnesses. Hooks emit composable effects at Enter/Send/Complete/Exit of each operation. |
| Credo, STITCH (full text unreadable; from abstracts) | Both treat harness pieces as typed, reusable primitives with scope and composition contracts, compiled per task. They work a layer above an ISA: closer to a standard library than an instruction set. |

| Source | Takeaway for the Harness ISA |
| --- | --- |
| [Quine (Mar 2026)](https://arxiv.org/html/2603.18030v1) | Agents as POSIX processes: mission in immutable `argv`, material on stdin, deliverable on stdout, diagnostics on stderr, outcome as exit status. Three state tiers (process memory, inherited env, filesystem). `exec` renews context while keeping identity and mission. Names its own limits: no "task-relative worlds" and no rollback that keeps lessons. |
| [AIOS (via review)](https://www.themoonlight.io/review/aios-llm-agent-operating-system) | Agent requests become syscalls through one scheduler; context manager can interrupt, snapshot, and restore an inference mid-flight. |
| [SGLang](https://arxiv.org/html/2312.07104v2) | Earliest small primitive set: `extend`, `gen`, `select`, `fork`, `join`. Exposing structure (forks, shared prefixes) is what lets the runtime optimize. |

Stop rule hit after 9 sources: the last two confirmed rather than changed the instruction table.

## Brainstorm

- **Context length as address size.** Worth pushing: RV32/RV64 became distinct bases rather than one superset. Should a 32K-context harness and a 1M-context harness be different HB variants, since programs written for one break on the other?
- **A "green card" for HB.** RISC-V's 2-page reference card is a forcing function. If HB can't fit on two pages, it's too big.
- **Illegal-instruction trap = unknown tool call.** RISC-V makes all-zero and all-one encodings illegal so stray jumps trap fast. Analog: the harness should treat a malformed or unknown tool call as a defined trap with a standard error, never silent recovery.
- **Macro-op fusion as a harness feature.** Advertise fused sequences (append + infer, fork + infer) in `CAPS` so programs know which patterns are cheap without changing semantics.
- **UNSPECIFIED as a first-class word.** RISC-V explicitly marks deliberately unconstrained behavior. HB should too, e.g. ordering of concurrent `STORE.PUT` from forks.
- **Quine's argv/stdin split might matter more for security than anything else here.** An injection-resistant mission slot is the one property hosted chat APIs don't give you cleanly today.
- **JAZ vs. this draft.** JAZ says one primitive is enough; this draft says 18. Both can be true: JAZ is the *language*, HB is the *machine*. Worth a short essay.
- **Conformance tests.** RISC-V has architecture test suites. A tiny HB conformance suite (trap kinds, preemption, mission immutability) could be the most useful artifact to build next.

## Parked questions

Raised during v0.2 research; deliberately not chased, per the stop rule.

- [ ] Read Credo and STITCH full text once readable copies are available; check whether their primitive catalogs imply missing base instructions.
- [ ] Read Andrew Waterman's RISC-V design thesis for more "why we left X out" rationale.
- [ ] Look at WASI as a precedent for a capability-based system interface standardized across runtimes.
- [ ] Check whether MCP's newer features (sampling, elicitation) already cover `INFER`-from-tool and `HUMAN.ASK`, and whether HB should align names.
- [ ] Look at Claude Code hooks, Pi extensions, and the Claude Agent SDK as de facto event models to compare with the four-phase design.
- [ ] Decide whether `RUN` stays in the base (also open in v0.2).

## Port test v0.2

Eleven real harness behaviors, written in v0.2 HB only. Result: 1 fully clean (H), 4 awkward (A, D, F, G), 3 not expressible (E, I, J), and 3 split, clean one way and not expressible another (B, C, K). The awkward and failing cases converge on ten v0.3 requirements, listed at the end.

### Results

| # | Behavior (source) | Expressed in v0.2 as | Verdict | Finding |
| --- | --- | --- | --- | --- |
| A | Block a dangerous bash command; model sees the reason and continues (Claude Code `PreToolUse`, exit 2) | `EVT.ON(TOOL.CALL, Send)` handler supplies a substitute result | Awkward | No standard tool-result error shape. "Abort" is ambiguous: abort the call, or the whole loop? |
| B | Refuse to stop until work is done (Claude Code `Stop` hook; `stop_hook_active` guard) | Program-owned loop: check before returning, `CTX.APPEND` a nudge, continue | Clean on a program loop; **not expressible** on `RUN` | `RUN` has no turn-end or stop event. A program loop needs none. Evidence for removing `RUN` |
| C | Back up the transcript before compaction (Claude Code `PreCompact`) | Derived compaction: `STORE.PUT(CTX.READ(ctx))` before rebuilding | Clean if the program compacts; **not expressible** for harness compaction | Invisible traps emit no events. "Invisible" must mean invisible to control flow, not to observers |
| D | Load project instructions at session start (Claude Code `SessionStart`, AGENTS.md) | `CTX.NEW(mission)` then `CTX.APPEND(STORE.GET("AGENTS.md"))` | Awkward | No session concept, and project policy enters as plain data, below the user's task in authority. Role alone can't label authority |
| E | Pause for approval, survive a process restart, resume with the answer (LangGraph `interrupt` + checkpointer) | `HUMAN.ASK` then `AWAIT`; extension `P` snapshots the context | **Not expressible** | The context can be snapshotted, but the program's own position (local variables, where it was waiting) cannot. Pending handles don't survive a restart |
| F | Rewind to an earlier step and try again (LangGraph time travel) | `CTX.SLICE(ctx, 0..k)` then continue | Awkward | Context rewinds cleanly; side effects of tools already run do not. Tools need a declared effect class |
| G | Hand off to a specialist agent that keeps the conversation (OpenAI Agents SDK handoff) | Contained trap on a `handoff` tool: `CTX.NEW(new_mission)`, `CTX.APPEND(CTX.READ(old))`, continue | Awkward | Immutable mission at the head of the context forces a copy: history re-enters as data and the cached prefix is lost, costing tokens on every handoff |
| H | Minimal four-tool coding loop (Pi) | Program loop over `INFER`, `TOOL.CALL`, `AWAIT`, `CTX.APPEND` | Clean | No `RUN` needed. Pi's `tool_call`/`tool_result` events map to `TOOL.CALL` Enter/Complete |
| I | A tool asks the client's model for a completion, or asks the user a question (MCP `sampling`, `elicitation`) | No route: tools are external and cannot call back into the harness | **Not expressible** | Tools need a re-entrant channel to a restricted subset (`INFER`, `HUMAN.ASK`) under the caller's capabilities and budget |
| J | Model emits malformed JSON or an unknown tool (every harness) | Hidden inside `INFER` | **Not expressible** | Known gap: `rejects[]`, illegal-call trap, repair policy |
| K | Run tool calls in parallel; user types a correction mid-generation (Claude Code, chat UIs) | Parallel: `AWAIT([TOOL.CALL(c) for c in calls])`. Interruption: no route | Parallel clean; interruption **not expressible** | No streaming `INFER`, no unsolicited human input, no defined partial result on cancel |

### New cases for v0.3 (unanswered)

Added after the loops session. Cases A–K plus these form v0.3's acceptance test; keep the same cases for later versions as a regression suite. Each states only the behavior to express.

| # | Behavior | Exercises |
| --- | --- | --- |
| L | A prime agent starts three subagents asynchronously, keeps talking with the user, and receives their results as interrupts; one finishes while the user is typing | Async receipts, interrupt delivery and priority, coalescing (R14) |
| M | Six coding subagents work tickets in one repo; two need the same file and must coordinate without the prime agent mediating | Named peer channels, leases on shared state, deadlock timeouts (R13) |
| N | A subagent hits an ambiguous requirement, asks its parent; the parent can't answer and forwards the question to the user | Routed asks, vertical trap routing, capability grants (R13) |
| O | Mid-session, switch the prime agent to a different model from another provider and continue without losing anything | Journal as truth, composed views, opaque model-bound blobs (R11, R12) |
| P | A long-running agent keeps a small working context, spills old material, and reloads a detail from 200 turns ago when it becomes relevant | Spill and reload, retrieval over the journal, working-set selection (R11, R12) |
| Q | Five sibling subagents launched together share one cached prefix; the scheduler keeps it warm through a two-minute pause | Shared canonical prefix, cache hints, keepalive (R12, R16) |
| R | A subagent crashes halfway through a task that already wrote two files; its supervisor retries it without duplicating or corrupting the writes | Effect classes, idempotency keys, compensations, supervision (R6) |
| S | The process dies while the prime agent waits three days on the user; on restart, the conversation resumes exactly | Durable waits on the user tool, resume by journal fold (R2, R11) |
| T | A self-adapting agent notices it performs the same five-step sequence on every invoice, rewrites it as a deterministic program, and uses that program from then on | Programs written by the model, JIT-style hardening, two entry points |

**Case X (added after v0.3, Recursive Language Models).** Given a corpus of 20 long documents in a store, answer a question that needs facts from three of them, using recursive decomposition and computation over the data in a persistent sandboxed REPL. The prime agent's context never holds raw document text. Measure tokens at each level of the tree. Exercises: `SPAWN` recursion, pass-by-name, a code-execution session tool, and (if present) ISA bindings inside the sandbox. Cases V and W follow.

**Case V (model roles).** A prime agent on a `plan` role delegates to `smol` subagents and a `vision` subagent; one provider rate-limits mid-run and its role falls back to the next model in the chain; the user switches profiles mid-session. Exercises R27.

**Case W (MemGPT-style agent).** Implement editable core memory, a FIFO message queue with a recursive summary, recall and archival search, a context-pressure warning, and heartbeat-style yielding, using only core instructions and a custom composer. Exercises composition, stores, and R28.

**Case Y (Pi extensions).** Re-implement one representative Pi extension from each functional category (see the Pi extensions survey in the raw notes) as programs, hooks, tools, and composers over HB: a permission gate, a custom compaction composer with cache warming, a provider plus role routing, a subagent team with file locks, an OpenTelemetry exporter, and a plan mode that removes editing tools mid-session. Record which needed something outside the ISA.

**Case Z (self-configuration for an edge device).** The same self-adapting agent, given an application, configures itself for two targets: a good laptop using hosted frontier models, and a low-resource edge device running small local models on one local inference server. It discovers what models and resources exist, chooses models per role, and picks a workflow to match: smaller views, more deterministic code, less parallel fan-out, no hosted cache warming. Compare tokens, wall-clock time, and task success between the two configurations.

### Three worked examples

**A: blocking a tool call.** The handler can substitute a result, but v0.2 doesn't say whether the model should see it as an error or a success, or what `Abort` would do instead.

```
EVT.ON(TOOL.CALL, Send, lambda e:
    Replace({"error": "blocked: destructive command"})   # shape not defined in v0.2
    if is_destructive(e.args) else Pass())
```

**E: approval across a restart.** Everything up to `AWAIT` is lost if the process dies, because the program's position is not state HB knows about.

```
plan = INFER(ctx, "big", {"schema": Plan})
h = HUMAN.ASK("Approve this plan?", ["yes", "no"])
answer = AWAIT([h])          # process restarts here: h, plan, and this line are gone
if answer == "yes": execute(plan)
```

The fix that fits minimalism is 12-Factor's: make every program a **stateless reducer**. All state lives in contexts and stores; the program is re-entered from the top on resume and reads where it was. Pending handles become serializable records rather than in-memory objects.

**G: handoff.** Two things fight: the mission is immutable and sits first, and caching rewards stable prefixes.

```
new = CTX.NEW(specialist_mission, caps, budget)
new = CTX.APPEND(new, "history", CTX.READ(old))   # full re-prefill, no shared prefix
```

If missions were a separate channel rather than the first entry, a handoff could fork the shared history and swap only the mission.

### v0.3 requirements

- [ ] **R1 Tool result shape.** Standard `{ok | error, content}` result, and defined meanings for aborting an operation vs. aborting the loop (A).
- [ ] **R2 Loop in the environment (revised).** Not "remove `RUN`": the agent loop is the environment's instruction cycle. Replace `RUN` with `SPAWN`, an environment call creating a model-driven context; end of turn is a call to the user tool or a HALT trap, and Claude Code's Stop hook is a contained handler for it (B, H; sketch 1).
- [ ] **R3 Invisible is not unobservable.** Every trap, including invisible ones, emits events (C).
- [ ] **R4 Authority labels.** Appended content carries an authority level, not just a role, and the label propagates into summaries and composed views (D; taint tracking).
- [ ] **R5 Stateless-reducer rule.** Subsumed by R11: program state is a fold over the journal (E).
- [ ] **R6 Tool effect class.** Tool specs declare read-only, idempotent, or side-effecting, with idempotency keys and optional compensations, so rewinds and retries know what is safe (F; sagas).
- [ ] **R7 Mission as a channel.** Retired as a separate fix: handoff dissolves under R11 and R12. Keep its goal, sharing cached history across missions, as part of R12 (G).
- [ ] **R8 Re-entrant tools.** Generalized by R13: tools hold a channel back to the caller's model and user, under the caller's caps (I).
- [ ] **R9 Decode outcomes.** `rejects[]`, illegal-call trap, and repair policy. Simplified by sketch 2: plain text with no call is itself illegal (J).
- [ ] **R10 Asynchronous input.** Generalized by R14 (K).
- [ ] **R11 Journal.** Write-ahead journal of every event (inference inputs and outputs, tool calls and results, messages, interrupts) as architectural state; durable, local, indexable for retrieval; opaque provider blobs kept but marked model-bound; retention and redaction policy (sketch 4).
- [ ] **R12 Composition primitives.** `COMPOSE(plan) -> ctx` from journal queries and stores; structural edits; per-entry token costs and cache state; spill and reload by reference; liveness hints; a shared canonical prefix across agents (sketch 5).
- [ ] **R13 Capability-gated IPC.** Channels as granted handles: RPC to parent, asks to the user (direct or routed up), spawning, named peer channels, shared state with leases. Every message journaled; every wait has a timeout or budget (sketch 6).
- [ ] **R14 Async calls and interrupts.** Every call returns a receipt; blocking is explicit; events delivered at instruction boundaries; coalescing before reinvocation; open: priority, masking, late results, cancel semantics (sketches 2 and 3).
- [ ] **R15 Identity and namespace.** Agents, channels, journals, stores, and tools are addressable names with generic verbs; an agent's namespace is its capability set (sketch 7).
- [ ] **R16 Cache hints and keepalive.** Non-semantic hints ("keep warm until T") that the scheduler may honor with keepalive calls or longer cache lifetimes when expected reuse justifies the cost (sketch 5).

R1–R10 came from the port test; R11–R16 came from the loops session. R4, R9, and R14 match questions already raised in the Harness Principles reference.

### v0.4 requirements (raised after v0.3)

v0.3 stays frozen as written. Gaps and ideas found after it go here, integrated from the raw notes at the end of the session on Oct 20. \*\*Principle adopted: inference is the tool of last resort\*\*, because it is slow, expensive, and unpredictable; prefer deterministic code wherever it can do the job, and let the model write code rather than repeat work through inference.

- [ ] **R17 Sandboxed code execution.** A standard extension defining a persistent code-execution session tool: `exec(session?, code) -> {session, output}`, one session per agent by default. The ISA specifies guarantees, not technology: no access beyond the owning agent's namespace and capabilities; CPU, memory, time, and network limits; the session's lifetime bound to its agent. Each executed cell is journaled. Large values belong in named stores; a lost session is re-created, rebuilt by replaying cells only if they were side-effect free, and otherwise reported to the agent (cases T, X).
- [ ] **R18 ISA bindings inside the sandbox.** A client library in the session exposes a subset of instructions (`read`, `put`, `query`, `infer`, `spawn`, `await`, `call`). Each function is a stub that marshals the call over a channel to the environment, the way libc wrappers trap into the kernel. The environment executes it as the owning agent: identity from a capability token injected at session creation, capability and budget checks, journaling with the session and cell as provenance. Nothing in the sandbox can raise its own privileges. Calls are async and return receipts, as everywhere else (cases T, X).
- [ ] **R19 One interface definition, generated twice.** The ISA is written once in an interface definition language. From it the environment generates both the tool definitions the model sees (first entry point) and the library stubs and documentation for sandboxes and programs (second entry point), so the two cannot drift. Precedents: protobuf and gRPC, WASI's WIT (which Pie uses for its inferlet bindings). The library's documentation ships as a shared `prefix` segment, cacheable across agents, or by progressive disclosure.
- [ ] **R20 Installed programs.** A program written in a session can be registered as a tool in a namespace: versioned, journaled, revertible, running in its own sandbox with declared capabilities. This is the case-T mechanism and the base of the self-adapting agent; R17–R19 are its prerequisites.
- [ ] **R21 Binding conformance.** The conformance suite exercises every instruction through both entry points and checks identical results and journal records, so a library stub can be verified against the tool path.

* [ ] **R22 Free text as a thinking HINT.** Text alongside calls is `think`: journaled, no architectural effect, shown to the user only if a composer chooses. A text-only response goes to the `text_as` allowance (treat as `user` or `return`, or `think` plus one nudge). Built-in `think` and `todo` tools. The default composer drops or summarizes past thinking. Rename the `INFER` result's `text` field to `think`. CPU parallel: RISC-V HINT encodings.
* [ ] **R23 Per-model codecs and `require_call`.** Each model registers a codec mapping its native encodings (thinking blocks, inline delimiters such as `<think>`, tool-call formats) to one outcome: `{think, calls, rejects, text}`. All thinking becomes the `think` journal kind; blobs a provider requires back are journaled as `opaque`. New `INFER` constraint `require_call`, implemented as forced tool choice where compatible, otherwise a system-prompt instruction, with the allowance as safety net.
* [ ] **R24 Eager dispatch.** Optional extension: start each tool call as soon as its block completes in the stream. Calls are journaled at dispatch; cancelling the parent `INFER` must define what happens to calls already started.
* [ ] **R25 Guardrails.** Verdict events `{verdict: allow | redact | quarantine | deny, reason, by}` linked to the content they judge; raw content and verdict both journaled, views use the guarded form. Ingress completeness rule: no external content enters the journal without passing its source's hooks. Egress guardrails sit on the `user` tool and other egress tools. Model-based guardrails run in parallel, with `CANCEL` as the tripwire. Depends on concrete authority labels.
* [ ] **R26 Provenance in one journal.** One journal per system with `agent`, `parent`, and `program` ids. Anything that changes or gates data is journaled as `transform` or `verdict` (component id, code version, inputs, output, reason); diagnostics stay in ordinary logs. The environment journals every hook effect automatically with its handler's identity. Recovery replays recorded outputs rather than re-running code. Repair tiers: deterministic codec repair, then invisible re-infer, then contained, then fatal.
* [ ] **R27 Model roles.** Programs name roles (`plan`, `smol`, `vision`); the environment resolves them through the namespace, with fallback chains as invisible traps and profiles as namespace mounts. `CAPS` describes each model: context length, modalities, reasoning support, cost, family. Role bindings may carry family-specific prompt variants. Reasoning effort is an `INFER` constraint. CPU parallel: heterogeneous cores (big.LITTLE) and accelerators.
* [ ] **R28 Query modes and context pressure.** `QUERY` supports exact, regex, and semantic modes (Letta's semantic-only history search failed exact-recall tasks in the JAZ evaluation). A named environment event signals context pressure so hooks and agents can react uniformly.

## Raw notes: loops session (Oct 19, evening)

Captured as said during the session and kept as the record. Integrated on Oct 19 into the Architecture sketch, Prior-art smorgasbord, and revised requirements above. Tags: **\[R\]** Rene's idea, **\[C\]** Claude's response or proposal.

### Two loops, and where the agent loop lives

- **\[R\]** CPUs loop in silicon; there is no assembly instruction for the fetch–execute loop. The port test also dropped `RUN`, but for a different reason (calling code implements the loop). Fundamental question: does the system provide the loop, or must calling code? What is an agent without a loop?
- **\[C\]** Two distinct loops. (1) Instruction cycle: when the model drives, the model is the CPU and its program counter; the model emits a tool call, the harness executes it, the result returns. That cycle *is* the agent loop, and it belongs in the silicon (the environment). (2) Program control flow: when deterministic code calls `INFER`, the model is a coprocessor or accelerator (RISC-V's term), and loops are ordinary program branches.
- **\[C\]** Matches Anthropic's split: workflows (code orchestrates model calls on predefined paths) vs. agents (model directs its own process). An agent is a model running as a processor.
- **\[C\] Proposed, not applied:** revise port-test R2 from "remove `RUN`" to "the loop belongs to the environment (HEI); replace `RUN` with hart creation (`SPAWN`, an environment call); model end-of-turn as a HALT trap." Claude Code's Stop hook becomes a contained handler for HALT. Both modes stay first-class: agent mode (model-driven hart) and workflow mode (program uses `INFER` as a coprocessor; may emulate the loop like an interpreter).
- **\[C\]** The five-instruction executable test then tests workflow mode and loop emulation; it no longer proves the loop is unnecessary.

### Subagents and the prime agent

- **\[R\]** A subagent that can't talk to any other agent or context, and can't spawn its own subagents, is essentially a nondeterministic function call. Context + instructions + goal go in; the model makes tool calls and halts; harness code takes an artifact (final output, a document) and hands it back by injecting it into the caller's context and invoking inference.
- **\[R\]** Define a **prime agent**: the one the user interacts with, which orchestrates subagents. Its loop may be no different, just with more varied tool calls.
- **\[R\]** The prime agent should delegate as much as possible to protect its own context. Ideally it is pure orchestration, does next to nothing itself, and is relied on only where deterministic code can't do the job.
- **\[R\]** Delegation raises context management: what of the prime's context passes to a subagent, via forking and cache reuse so the subagent doesn't re-prefill what the prime already filled.
- **\[R\]** Conclusion while talking it through: there is always at least one master loop running, analogous to the silicon. A subagent call is still a function call within it.
- **\[C\]** The master loop is an **event loop**; agents are threads it schedules, not loops themselves. Events (tool result, user message, timer, subagent done) make a context runnable; the loop invokes the model on it. A subagent "running" is the scheduler revisiting its context until it halts. Same picture as RISC-V harts advanced by the environment.
- **\[C\]** Delegation is a token decision per call. Fork: inherit the prime's context at cached-prefix rates, but those tokens still fill the window and old chatter can distract. Fresh: small and clean, no cache benefit, costs a written brief. Strengthens R7 (mission as a channel), so forking with a different mission keeps the cache.

### Everything is a tool call, including the user

- **\[R\]** Frame everything as a tool call, including user interaction. When the prime agent is done, it emits a user-interaction tool call whose argument is the output to show; the tool's response is whatever the user types. The loop waits indefinitely on a tool call that needs the user to complete. Analogous to a REPL waiting on input.
- **\[C\]** Precedent: 12-Factor "contact humans with tool calls," Claude Code's ask-the-user tool, LangGraph `interrupt`. New here: making it universal, so even a final answer is a user-tool call.
- **\[C\]** Consequences: every model output decodes to tool calls (no text-vs-call path), so plain text with no call becomes an illegal instruction, simplifying R9. The user is one more device. Halting stops being special: blocked on the user tool is just slow I/O, which makes durability (R5) mandatory, since "slow" can mean days.
- **\[C\]** Wrinkle: models are post-trained to end turns with plain text. The harness can treat final text as an implicit user-tool call, or prompt and train against it. Decode-layer detail, not an ISA problem.

### Synchronous vs. asynchronous calls and interrupts

- **\[R\]** Do tool calls block the caller or not? If the prime kicks off a subagent without blocking, it continues; the harness picks up the call, starts the subagent, and later invokes the prime again, immediately or when an event such as a tool response arrives.
- **\[R\]** You never really interrupt an inference call, especially on a hosted model; cancellation may have a place. An interrupt event (such as an async call returning) gets handled by the harness, possibly with deterministic processing, and eventually the prime model is invoked again with something like a non sequitur injected into its context, interrupting what it was doing.
- **\[C\]** Interrupts arrive at instruction boundaries, and the inference call is the atomic instruction, just as CPUs take interrupts between instructions. Events for a busy context are held until its call completes, then injected before the next call. Cancel = killing the instruction; the tokens already spent are lost.
- **\[C\]** Correction to the mid-stream idea: hosted APIs end the turn at the tool calls; the model doesn't keep generating after them. Async still works: an async tool returns a **receipt** immediately ("subagent #7 started"), the loop can reinvoke the prime at once, and the real result arrives later as an interrupt.

### Open questions raised

- Interrupt ordering and priority: if the user types while three subagents finish, what does the prime see first? Does user input outrank subagent results?
- Masking: can a context defer interrupts during a critical sequence?
- Interrupt storms: coalesce many completions into one batched event (a token optimization)?
- Ownership of late async results: if the prime has moved on, inject or drop?
- Fork vs. fresh context: who decides per delegation, and on what token estimate?

### Shells and subshells

- **\[R\]** Shell analogy: at a terminal you give instructions that run as small programs; a shell script is full of more shell instructions, and each one runs in a newly spawned shell. That's where "shell" earns its name. Good parallel for subagents: the context is like the arguments, the instructions are like the program it runs. Scripts call other shell programs with arguments, giving branching, nesting, and returns.
- **\[C\]** The mapping extends: interactive shell = prime agent (a REPL waiting on the user); script = skill program; subshell = subagent; exported environment = scoped capabilities and variables inherited by children; a subshell's changes don't leak to its parent = fork isolation; exit status = structured outcome; pipes = chaining subagents' outputs into inputs.
- **\[C\]** Job control supplies a ready vocabulary for async subagents: `&` (start without blocking, get a job id = receipt), `wait`, `jobs`, `fg`, `kill` (= `SIGNAL`), and the shell's notice when a background job finishes (= interrupt at the next prompt, an instruction boundary). Quine already builds on this model.

### Rich subagent communication

- **\[R\]** Beyond the isolated subagent: allow, when wanted, a subagent to ask the user a question, ask its calling agent a question, spawn its own subagents, and talk to other subagents outside its own call chain.
- **\[R\]** Example: the prime agent kicks off a dev team of about six coding subagents. If two have tickets touching the same code, they need to coordinate so they don't edit one file at the same time, as real engineers do. The architecture should support peer-to-peer communication.
- **\[C\]** This turns the call tree into a process graph with IPC. It needs addressing (agent identities, which argues for a real `SPAWN` handle), channels or mailboxes, and delivery as interrupts at instruction boundaries.
- **\[C\]** Make every channel a capability. Default is the isolated function-call subagent; parents grant more: ask-user, ask-parent, spawn, peer messaging. The simple case stays simple.
- **\[C\]** Asking the user from a subagent is a routing question: directly, or as a trap routed up through the parent, which may answer itself, batch questions, or forward them (RISC-V vertical traps). The parent's grant decides.
- **\[C\]** Two coordination styles, both needed: messaging (negotiation, "I'm changing the auth module, hold off") and shared-state primitives (a lease or lock on a store key or file, compare-and-swap). Shared state is far cheaper in tokens than conversation, and is how real dev teams mostly coordinate (branches, locks, tickets). CPU parallel: atomics and the memory model.
- **\[C\]** Risks: token blow-up from chatter, deadlock (A waits on B waits on A, so every wait needs a timeout or budget), and harder tracing. Precedents: MCP sampling and elicitation (a tool reaching back to the caller's model or user), and the Agent2Agent (A2A) protocol for agent-to-agent messaging.

### Context management as register allocation

- **\[R\]** Context assembly before each inference call is the analog of register allocation in a compiler. The default today is append-everything, which brings bloat, lost-in-the-middle, and cost. Existing approaches: Letta treats context like RAM; a recent project lets the model prune its own KV cache. The prime-agent-as-orchestrator idea is the same concern: keep the prime's context manageable.
- **\[R\]** As a compiler manages registers, the calling code written against our API will implement context-management logic. The API must support that.
- **\[C\]** What a "context compiler" needs from the API, checked against v0.2:
  - **Structural edits, not just append.** v0.2 has `READ`, `SLICE`, `FORK`, `NEW`, enough to rebuild a context but not to replace one entry with a summary in place. Rebuilding loses the cache.
  - **Cost visibility.** Token count per entry and which prefix is currently cached, so the allocator can price decisions (a register allocator knows spill costs).
  - **Spill and reload by reference.** Evict an entry to a store, leave a handle, reload on demand: Letta-style paging.
  - **Liveness hints.** Mark entries pinned, ephemeral, or evict-after-use (tool-result clearing is one case).
  - **Model self-management.** Expose the context operations as tools so the model can prune its own context. The two-entry-points principle already allows this.
- **\[C\]** Core tension: editing the middle of a context invalidates the cache for everything after it. Allocation must be layout-aware: stable content first, volatile content last, and edits batched. This is the cache-aware form of register allocation and the main place token optimization happens.

### Pending

- **\[R\]** One more idea to raise; forgotten mid-conversation. Recalled: see "Durable journal, ephemeral contexts" below.

### Clarification: "two entry points"

- **\[R\]** Asked whether "two entry points" means the API is implemented on the provider side and called from the calling side.
- **\[C\]** No, that's a different idea: the two *faces* of the API (the upward face binds callers, the downward face binds implementers). "Two entry points" (v0.1/v0.2 principle 1) means the same primitives are reachable by two kinds of caller: the **model**, by emitting tool calls (the nondeterministic trap path), and **deterministic code** such as skill programs or harness logic, by ordinary library calls. Example: `CTX.SLICE` can be a tool the model calls to prune its own context, and a function a context-compiler program calls. Same operation, same semantics.

### Durable journal, ephemeral contexts

- **\[R\]** Users treat the context as sacred: afraid to compact, every token precious. Pi relies on an append-only JSONL journal. Claude Code keeps session transcripts in dot folders under home, presumably spanning compaction events.
- **\[R\]** When composing a context for an inference call, the prior turn's context may not hold the full history, because not all of it is relevant. Keep everything anyway; the harness could run a subagent that does retrieval over the complete history and puts the relevant parts into another agent's context, such as the prime agent's. That requires all the details to exist.
- **\[R\]** So: absolutely everything lands in a durable store (files or a database, on the local system, not a provider's KV cache in a cloud we don't control). Every context is **intentionally composed** from that store. The context is no longer sacred; it is always ephemeral. You can change agent or model, or fork, because everything is somewhere durable.
- **\[R\]** Durability = portability = inherent handoff. Handoff shouldn't exist as a concept, because every turn is a handoff from the previous one.
- **\[R\]** Not a mandate that everything be built this way, but the architecture must not preclude it.
- **\[C\]** This is event sourcing. The journal is the source of truth (a write-ahead log: every event lands before it takes effect); a context is a **view**, a projection over the journal and stores, rebuilt as needed. CPU analog: contexts are registers and cache; the journal is main memory and disk. A provider's KV cache is a cache of a cache.
- **\[C\]** Consequences:
  - The stateless-reducer rule (R5) falls out: program state = fold over the journal.
  - Compaction stops being lossy. The source stays; compaction is just choosing a smaller view.
  - Handoff dissolves, resolving port-test case G: switching agent, mission, or model means composing a new view.
  - Append-everything is just the trivial view (the whole journal), so this design is a superset of today's default, not a replacement.
  - Retrieval over history is a query over the journal, so the journal must be indexable (a `STORE.QUERY` target).
- **\[C\]** Possible ISA shape: the journal is architectural state every instruction writes to, like memory; contexts are built with something like `COMPOSE(plan) -> ctx` from journal queries and store reads. That might shrink the Context group rather than grow it.
- **\[C\]** Tensions:
  - **Cache.** Recomposing each turn can forfeit prefix caching. Composition policy should keep a stable prefix and vary the tail. The cache is an optimization; the journal is truth.
  - **Opaque provider state.** Some providers return encrypted or opaque reasoning items that must be sent back verbatim. Journal them as opaque blobs, but they only replay to the same model, a limit on full portability.
  - **Size and sensitivity.** Keeping everything means retention policies, secret redaction, and access control on the journal.

### Pipes and RPC for agent communication

- **\[R\]** Agreed on two entry points and on durability and state retention.
- **\[R\]** Subagent-to-subagent communication brings to mind POSIX pipes and RPC; both seem applicable.
- **\[C\]** Mapping: subagent-as-tool is **RPC** (request, typed response). A subagent asking its parent a question is an RPC upcall. A chain of agents feeding each other is a **pipeline** (one-way stream with backpressure). Peers outside one call chain are **named pipes or message queues**: unrelated processes rendezvous on a name. Coordination over shared files uses locks (`flock`), matching the lease idea for the dev-team case.
- **\[C\]** In Unix, a pipe's file descriptor handed to a child is literally a capability: it can use only the channels it was given (Capsicum formalizes this). That fits "every channel is a capability": channels are handles, granted by the parent.
- **\[C\]** Every message on every channel is a journal event, so IPC is durable and replayable like everything else.

### Cache-aware composition

- **\[R\]** Managing the hosted KV cache is a major consideration. Keep inference context concise, but don't throw away cost and time savings by re-prefilling what an existing KV cache segment already holds. Structuring the context for every call is part of the optimization, analogous to a compiler managing CPU register pressure.
- **\[C\]** Composition has two goals that pull against each other: a small window (quality, cost) and maximal reuse of cached prefixes (cost, latency). Cached reads are typically billed at a large discount and return faster.
- **\[C\]** Design move: give all agents in a system a shared canonical prefix (base instructions, tool definitions), so siblings and the prime agent hit the same cache. This is the shared-library pattern. Agent-specific mission and material go after it.
- **\[C\]** Provider caches match exact prefixes and expire after minutes. Cache warmth therefore becomes a scheduling input: running sibling subagents close together in time saves tokens. The event loop, not just the composer, participates in token optimization.

### Session wrap-up

- **\[R\]** This covers most of what was planned for the brainstorm. Next: integrate raw notes into the rest of the workbook.
- **\[C\]** The session's main shift, in five parts: (1) one event loop in the environment schedules contexts as harts; (2) everything is a tool call, the user included; (3) agent communication is capability-gated IPC (RPC, pipes, named channels, shared state with locks); (4) the journal is the source of truth and contexts are composed views; (5) composition is cache-aware register allocation, measured in tokens. These reshape v0.3 more than the port-test requirements alone.

### Keepalive, everything-is-a-file, async by default

- **\[R\]** Keep-alive heartbeat calls: an inference call that is effectively a no-op, made only to keep a KV cache segment alive.
- **\[C\]** Valid, and it belongs to scheduling policy rather than the instruction set: a timer event in the event loop. Each ping costs a cached-prefix read plus a tiny output, so ping only when expected reuse outweighs that. Some providers sell longer cache lifetimes, which may beat pinging. A cache hint such as "keep warm until T" lets the environment choose.
- **\[R\]** Unix (and Linux) frames everything as a file. "Everything is a tool call" is a loose analogy to that; worth checking for further ramifications.
- **\[C\]** Unix's power came from two things together: a **uniform interface** (open, read, write, close) and a **namespace** (paths), so generic tools work on everything. The ramification: tool calls need the same, a small set of generic verbs and addressable names, so cross-cutting programs (journaling, permission checks, mocking, replay, tracing) apply to every tool uniformly. Agents, channels, journals, stores, and contexts would all be addressable names. MCP resources already use URIs.
- **\[C\]** Plan 9 took it further with per-process namespaces. A **per-agent namespace** (what an agent can see and name) is a clean way to express its capability set, and it answers Quine's open question about task-relative worlds: a child's world is the namespace its parent mounts for it.
- **\[R\]** Every tool call should have the option of being asynchronous, and async should probably be the default. Either the harness or the model must explicitly say a call blocks.
- **\[C\]** Agree, with one cost to manage: every reinvocation of a model is another inference call, so naive async-by-default multiplies calls and tokens. The environment should **coalesce**: wait briefly for fast completions before reinvoking, like interrupt coalescing or Nagle's algorithm in TCP. Blocking stays explicit (`AWAIT`).
- **\[R\]** We're in new territory with language models. Don't confine the design to ISA or POSIX paradigms; the past is a smorgasbord. Survey every established practice that fits: IPC, shells, and anything else.
- **\[C\]** Done in the "Prior-art smorgasbord" section above.

## Raw notes: after v0.3 (Oct 19, late)

Integrated on Oct 20 into "v0.4 requirements" (R17–R28 and the adopted principle) and port-test cases U–X. Kept as the record. v0.3 unchanged; the handoff gained case U only.

### How hosted models end a response

- **\[R\]** Claude said at one point that hosted models end with tool calls, and at another that they end with natural language for the user. These seemed contradictory, with different implications for async calls and for parsing a stream to start tool calls early.
- **\[C\]** Both are true; the earlier wording was imprecise. A single response ends one of two ways: with tool calls (more work to do), or with prose (the model thinks it's done). Within one response the typical order is optional prose, then tool calls, then stop. The model does not continue producing prose after its tool calls in the same response; it resumes after the results come back.
- **\[C\]** Streaming still allows **eager dispatch**: start each tool call as soon as its block completes in the stream, before the response ends. The gain is modest, mostly with parallel calls. It fits v0.2's extension `S`. It doesn't break `INFER` atomicity if each call is journaled when dispatched, but a later `CANCEL` of that `INFER` must then decide what to do with calls already started.

### Free text as thinking out loud

- **\[R\]** Under "everything is a tool call," provide a scratchpad or think-out-loud tool for throwaway reasoning that helps reach a conclusion, and a to-do list tool.
- **\[R\]** Possibly make one exception to the rule: any output that is not a tool call is treated as thinking out loud. That makes free text legal, gives the harness a valid way to handle it, and avoids rejecting output just because it wasn't wrapped in a call.
- **\[C\]** Precedents: Anthropic's published "think" tool for pausing during complex tool use, and Claude Code's to-do list tool. Reasoning models also have native thinking blocks, a separate channel the journal could record as a `thinking` kind.
- **\[C\]** CPU parallel: RISC-V defines whole classes of encodings as HINTs, which execute as no-ops, instead of making them illegal. Free text as "think" is a HINT with a payload: journaled, no architectural effect, never shown to the user unless a composer chooses to.
- **\[C\]** It matches observed behavior: the prose models emit before tool calls ("Let me check the config") is already thinking out loud. Text alongside calls becomes `think` cleanly.
- **\[C\]** The hard case is a text-only response. A model that is done usually writes its final answer as prose with no call. If that becomes `think`, the answer never reaches the user and the environment re-invokes a model that believes it finished, costing a turn or looping. Proposed rule: text alongside calls is `think`; a text-only response goes to the `text_as` allowance (treat as `user` or `return`, or record as `think` plus one nudge to call `user`).
- **\[C\]** Token cost: think text is output tokens, and it re-enters later views unless a composer elides old thinking. The default composer should drop or summarize past think entries.
- **\[C\]** First build is unaffected: v0.3's `INFER` result already carries `text` beside `calls`, and text-only handling is the `text_as` allowance. Candidate for v0.4: rename `text` to `think` with the HINT semantics above, and add built-in `think` and `todo` tools.

### Forcing the user tool; parsing thinking delimiters

- **\[R\]** The other default for non-call output is to treat it all as user output. Would a simple system-prompt instruction ("output for the user must be wrapped in a `user` tool call") be enough to address models ending turns in natural language?
- **\[C\]** Mostly, and there's a stronger lever: most major APIs can **force** a tool call per response ("any tool" or "required" tool choice), which makes text-only responses impossible at the API level. Caveat, as far as known: on some providers, forced tool choice is incompatible with extended thinking, and forcing may cost some reasoning quality. Proposed layering: an `INFER` constraint `require_call`, implemented by the decoder as forced tool choice where compatible and as a system-prompt instruction otherwise, with the `text_as` allowance as the safety net for rare slips.
- **\[R\]** Some providers may detect special think-start and think-end tokens, let the model see its reasoning autoregressively, and scrub it from the response. If the provider does that, the harness needn't offer think space. If not, and the model emits thinking anyway, the harness should support it, possibly by parsing enclosing special tokens the way it parses tool-call JSON.
- **\[C\]** That is exactly how it works today, in three styles: closed reasoning models return thinking in separate blocks or hidden reasoning items (sometimes encrypted or signed, and sometimes required to be passed back in tool-use loops); open-weight reasoning models emit inline delimiters such as `<think>…</think>`, which serving stacks split out with model-specific reasoning parsers; non-reasoning models just write prose before their calls.
- **\[C\]** Architectural answer: a **per-model codec** at the decode layer. Each model registers how it encodes calls and thinking (native blocks, inline delimiters, JSON formats), and the codec maps all of them to the same architectural outcome: `{think, calls, rejects, text}`. Thinking from any source (native block, delimiters, a `think` tool call, leftover prose) becomes one `think` journal kind; required pass-back blobs are journaled as `opaque`. CPU parallel: one ISA, several encodings (RISC-V standard and compressed). This answers question 8 in the Harness Principles reference (the model encoding layer).

### Guardrails on input and output

- **\[R\]** Does the architecture support guardrails? Two faces: output (the model mustn't say something to a user it shouldn't) and input (evaluate content as it enters the system: prompt injection in user prompts, malicious material in results from curl, network, or web search tools). Input checks shouldn't happen at composition time; anything available to compose should already have passed a guardrail.
- **\[C\]** Mostly supported through events, with four gaps.
- **\[C\]** Output: because everything is a tool call, the `user` tool is the **single egress point** to the user. An output guardrail is a handler on `ON(user, Enter)` that can allow, rewrite, or deny before display. Other egress tools (email, HTTP POST, file writes) get the same treatment, which also covers data exfiltration. Tool-argument checks are `ON(CALL, Enter)`, already case A.
- **\[C\]** Input: every ingress arrives as a completion or event (user replies, tool results, subagent returns, channel messages), so input guardrails are `ON(..., Complete)` handlers. Agree that evaluation belongs at ingress, not composition. But composition must still **honor** the verdict without re-evaluating it: label at ingress, enforce at use (taint tracking).
- **\[C\]** Model-based guardrails are themselves `INFER` calls (workflow mode) costing tokens and latency. Async-by-default lets them run in parallel with the main inference; a tripwire is a `CANCEL` on the main call, the pattern OpenAI's Agents SDK uses.
- **\[C\] Gaps in v0.3:**
  1. **No verdict shape.** Proposed: `{verdict: allow | redact | quarantine | deny, reason, by}`, recorded as its own journal event referencing the content it judges.
  2. **Ordering with the write-ahead rule.** Proposed: the raw content is journaled (kept for audit and forensics), the verdict is journaled as a separate linked event, and views use the guarded form. The journal is never rewritten.
  3. **No ingress-completeness rule.** Proposed: no external content enters the journal without passing the ingress hooks for its source. Architectural, not optional.
  4. **Authority labels are only sketched.** Input guardrails depend on them: retrieved and tool-returned content is low authority, and instructions found in it are data, never policy.
- **\[C\]** First build: case A already exercises a pre-call guardrail. An input-guardrail case (injected instructions in a mock web-fetch result, quarantined and composed as labeled data) would be a cheap addition to the acceptance tests.

* **\[R\]** Agreed: add the input-guardrail case to the handoff's acceptance list. (Done: case U.)

### One journal, provenance of harness code

- **\[R\]** Inference and tool-call activity go in one journal for the whole system, subagents included, not a cloud of per-agent files. Each agent gets its own ID, like a PID, and tool calls carry the calling agent's ID.
- **\[R\]** Calling code (output processing, guardrails) is deterministic code, normally covered by ordinary logging. Logs are also append-only journals; the difference is how we regard them. The agent journal is precious; program execution logs are ephemeral and only for diagnosing errors.
- **\[R\]** But harness code can alter the data that ends up in the main journal: a guardrail rewriting or blocking content, a JSON repair. We'd see evidence that something changed, but not which code changed it or why. Use cases: forensics (how did a bad message reach the user?), and durability and handoff.
- **\[R\]** Example: on a malformed tool call, the first tier is deterministic repair (add a missing closing brace); if the arguments are unrecoverable, the second tier sends it back to the model with another inference call asking it to fix it, which already lands in the main journal.
- **\[R\]** Leaning toward putting all data-affecting harness activity in the main journal: one journal for the entire system. Unsure whether it affects the API.
- **\[C\]** Agree on one journal. v0.3 already scopes it to "a system of agents" with an `agent` field on every event; add `parent` for lineage, and a `program` id for workflow-mode code.
- **\[C\]** The dividing line: **provenance goes in the journal; diagnostics go in logs.** Test: does this event change or gate data that flows through the system? If yes, it is journal material: transforms, repairs, guardrail verdicts, denials, composition plans. Timings, debug traces, and stack traces stay in ordinary, ephemeral logs.
- **\[C\]** Proposed event kinds: `transform` (`by`: component id and code version; input refs; output; reason) and `verdict` (from the guardrail notes). The JSON-repair example becomes a chain: raw `infer.result` → `transform` by the codec's repairer ("added missing brace") → `call`; or, if repair fails, → `infer.request` asking the model to fix it. Smorgasbord parallel: data lineage and provenance records (W3C PROV), database audit logs.
- **\[C\]** Recording outputs, not just code identity, keeps durability independent of code versions: recovery replays recorded outputs instead of re-running the transforms, as record-and-replay debuggers do. Code identity is for forensics.
- **\[C\]** API impact is small: hooks act through structured effects (rewrite, deny, replace), so the **environment can journal every effect automatically**, with the handler's identity. No new instruction. Transforms done inside a workflow program, outside hooks, are visible by their outputs (whatever the program passes to the next instruction); the program can add the reason with `PUT` on the journal as a `transform` note.
- **\[C\]** Repair policy gains a deterministic first tier: codec-level repair → invisible re-infer → contained → fatal, each tier journaled. v0.4 candidate.

### Different models for different tasks

- **\[R\]** Using different models for different tasks is somewhat like a CPU delegating to an ALU or another special processor. Oh My Pi is a good test case: it identifies multiple agent roles and supports a different model for each.
- **\[C\]** Checked: Oh My Pi (omp, built on Pi) routes work through **model roles** (`default`, `smol`, `slow`, `vision`, `plan`, `designer`, `commit`, `task`), each mapped to a provider/model with an optional reasoning-effort suffix, with fallback chains and switchable profiles ([omp-model-profile](https://github.com/tlarevo/omp-model-profile), [config generator](https://inventivehq.com/tools/oh-my-pi-config-generator)). The related oh-my-opencode goes further: each agent's system prompt is tuned to a **model family**, and putting the wrong family on an agent degrades it; some agents detect the family and switch prompts ([agent-model matching guide](https://gitea.gravitywell.xyz/sapient/oh-my-openagent/src/commit/d9ee0d9c0db8a5743968fe12acae57b47b7a93c3/docs/guide/agent-model-matching.md)).
- **\[C\]** v0.3 supports the mechanics: `INFER` takes a model per call, `SPAWN` a model per agent. Missing:
  1. **Role indirection.** Programs should name a role (`plan`, `smol`), not a concrete model; the environment resolves it. Fits sketch 7: models are names in the namespace, and a profile is a namespace mount.
  2. **Fallback chains.** On rate limits or outages, retry on the next model in the role's chain: an invisible trap, journaled.
  3. **Model descriptions in `CAPS`.** Context length, modalities, reasoning support, cost per token, family.
  4. **Family-specific prompts.** A role binding pairs a model with a prompt variant; plans may carry per-family mission variants, and the composer picks one. Note that switching families also forfeits cache, since providers don't share it.
  5. **Reasoning effort** as an `INFER` constraint.
- **\[C\]** CPU parallel, sharper than ALU: **heterogeneous cores** (ARM big.LITTLE): performance and efficiency cores with a scheduler choosing per task, plus accelerators for specific workloads (vision as a GPU). Routing is the scheduler's or caller's decision, and all three styles are expressible: static profile (omp), per-agent binding (oh-my-opencode), or a dynamic router where a cheap model picks the model for each step. Ties back to compiler instruction selection.
- **\[C\]** Proposed port-test case **V**: a prime agent on a `plan` role delegates to `smol` subagents and a `vision` subagent; one provider rate-limits mid-run and the role falls back; the user switches profiles mid-session.

* **\[R\]** No need to build a role map for Claude Code; the architecture supporting it is enough.

### Letta (MemGPT) as a test case

- **\[R\]** Letta, first called MemGPT, leans on context-as-RAM and divides the context window into functional segments: system prompt, user context, and possibly tasks and tool calls. The model manipulates its own context, perhaps emitting calls from a to-do list or call stack and purging the caller's to-do once the response arrives, plus compaction by summarizing user context. All of this is context composition; the architecture probably covers it, but wants an evaluation.
- **\[C\]** From memory of the MemGPT paper (Packer et al., 2023) and Letta: main context = read-only system instructions + editable **working context** (core memory blocks such as persona and human, edited by the model through memory functions) + a **FIFO message queue** headed by a recursive summary. External context = **recall storage** (full message history, searchable) and **archival storage** (vector store). A memory-pressure warning fires near the context limit; eviction flushes old messages to recall storage and updates the summary. **Function chaining via heartbeats**: the model must request continued execution, or it yields to the user; timed heartbeats and system alerts are also events. There is no call stack in the paper as far as recalled; heartbeat chaining is the closest thing.
- **\[C\]** Mapping to v0.3:

| MemGPT / Letta | v0.3 | Fit |
| --- | --- | --- |
| System instructions | `prefix` and `mission` segments | Clean |
| Working context (core memory blocks, model-editable) | `store(...)` segments plus `PUT` exposed to the model as a tool | Clean (two entry points) |
| FIFO queue with recursive summary | `events(...)` plus `ref(...)` summaries, via a composer | Clean |
| Recall storage | The journal plus `QUERY` | Clean; the journal is a superset |
| Archival storage | A store with vector `QUERY` | Clean |
| Memory-pressure warning | Composer reads the view report and injects a warning | Works; no standard environment event |
| Eviction and flush | Spill with `ref` segments | Clean, and non-destructive |
| Heartbeat chaining | Loop policy: MemGPT yields unless continuation is requested; v0.3 continues unless `user` or `return` is called | Both expressible |
| Timed heartbeats, system alerts | Scheduler timers and interrupts | Clean |

- **\[C\]** Verdict: Letta is expressible as one composer, three or four stores, and a handful of memory tools. Strains and lessons:
  1. **Model-driven vs. program-driven composition.** Letta lets the model edit its own memory; v0.3 defaults to composer programs. Both work, and hybrids work because composers can read model-edited stores.
  2. **Cache cost of self-editing memory.** Core memory sits near the top, so every edit re-prefills everything after it. v0.3's view report makes that cost visible; Letta pays it silently. Placement of editable blocks becomes a token decision.
  3. **Search modes.** The JAZ paper beat Letta on StuLife's far-recall tasks partly because Letta's history search couldn't do exact substring matching. `QUERY` should support exact and regex modes, not only semantic search.
  4. **No standard "context pressure" event.** Minor gap; a named environment event would let hooks and agents react uniformly.
- **\[C\]** On the remembered call stack: whether or not MemGPT had one, it is a good composer strategy. Push a frame for a subtask, pop it when the result returns, and evict the caller's to-do entry (a liveness hint). Subagents give this structure naturally: each fresh context is a new stack frame.
- **\[C\]** Proposed port-test case **W**: implement a MemGPT-style agent (editable core memory, FIFO queue with recursive summary, recall and archival search, pressure warning, heartbeat-style yielding) using only v0.3 instructions and a custom composer.

* **\[R\]** Correction: it was the MemGPT *message queue* being remembered, not a call stack.
* **\[R\] Principle: inference is the tool of last resort.** It is slow, expensive, and unpredictable, three qualities you don't want in any system, and they sit at the heart of modern AI. MemGPT does the opposite: much of the context management it hands to the model could be done by harness code.
* **\[C\]** Agree, and v0.3 already leans this way (composer programs by default, model-driven memory as an option). Candidate for promotion to a stated design principle alongside minimal decomposition and the token metric. It also connects to JIT-style hardening (case T) and to the prime agent relying on deterministic code wherever it can.

### Recursive Language Models (RLM) as a test case

- **\[R\]** RLM (unsure of origin) has two ideas. **Recursion:** for a huge corpus, spawn a subagent per book, which spawns one per chapter, then per section; results return up the tree, so the prime context holds only what's relevant and everything irrelevant passes only through leaf contexts. **The environment holds the data:** everything the model generates goes straight into a Python REPL, so large data can live in variables and never enter any model's context. Generalized: the architecture uses the environment, not the context window, for data handling. Other architectures do the same by writing data to files.
- **\[R\]** RLM framed all model output as a call to a Python tool, which "everything is a tool call" covers; subagents should cover the recursion. Asks how well the architecture supports the paradigm.
- **\[C\]** Origin: Zhang, Kraska, and Khattab at MIT, "Recursive Language Models" (arXiv 2512.24601), already cited in the JAZ paper. JAZ's `invoke` is a generalization of it.
- **\[C\] Recursion: supported.** `SPAWN` with fresh plans per book, chapter, and section; scoped budgets shared down the tree; async fan-out with coalesced returns; cheap model roles at the leaves. Many of RLM's recursive calls are plain model calls on a chunk rather than full agents; that's workflow-mode `INFER` on a small view, cheaper than `SPAWN`.
- **\[C\] Environment data handling: supported in principle.** Data lives in stores and the journal under names; plans and subagent specs pass **names, not contents** (JAZ's pass-by-reference). A subagent can be handed `/corpus/book3` without the parent ever reading it.
- **\[C\] The real gap: computing over data without reading it.** RLM's power is that the model writes code to grep, slice, count, and chunk data in place. v0.3 only offers `READ` and `QUERY`, a fixed query language. Full RLM needs a sandboxed code-execution tool (v0.2's extension `E`) with **ISA bindings inside the sandbox**: model-written code that can call `READ`, `QUERY`, `INFER`, and `SPAWN` under the agent's capabilities and budget. In RLM terms, that's the REPL with `llm_query` bound in.
- **\[C\]** That converges with case T: RLM's model-written code is a *temporary* program run once; case T's is a program *installed* for reuse. One mechanism (sandboxed programs with ISA bindings) serves both, which makes it the most important v0.4 item.
- **\[C\] Durability tension.** REPL variables are in-memory state, which violates the journal rule: lost on restart. Two resolutions: back large values with named stores (files), leaving the REPL to compute over them; or record the code each REPL step ran and replay it on recovery, valid only for side-effect-free code. Prefer the first, which matches the files-instead-of-variables variant mentioned above.
- **\[C\]** Proposed port-test case **X**: given a corpus of 20 long documents in a store, answer a question needing facts from three of them, using recursive decomposition and in-sandbox computation; the prime agent's context never holds raw document text. Measure tokens at each level of the tree.

### Sandboxes, persistent REPLs, and ISA access from code

- **\[R\]** Added case X to the port test. A one-shot Python tool (run a script, return output) isn't how RLM works: RLM needs the REPL to persist across calls. Proposal: a harness-provided Python tool whose first call starts a runtime and returns its id, so each subagent gets its own REPL and variables. That can be harness code as a custom tool. The agent can still call `SPAWN` and other instructions directly as tools, without tunneling through Python. Does the architecture support this? And what is a sandbox: just a Python runtime with an active REPL?
- **\[C\] What a sandbox is.** Not the interpreter; the **containment** around it. An execution environment where code runs with restricted filesystem, network, and syscall access and bounded CPU, memory, and time, so model-written code can't harm the host or exceed its grants. Implementations: containers, microVMs, gVisor, OS sandboxing (seccomp, bubblewrap), WebAssembly (Pie runs inferlets in Wasm), Anthropic's sandbox runtime. RLM's REPL is a persistent interpreter *inside* such a boundary.
- **\[C\] Persistent REPL as a session tool: yes.** `python(session?, code) -> {session, output}`, as Jupyter kernels and hosted code interpreters work. Each subagent gets its own session.
- **\[C\] Design (a), the proposal: model drives both.** The model calls `python` to process data and calls `SPAWN` directly to recurse. Supported in v0.3 with one bridge: REPL variables aren't names in the ISA namespace, so data a subagent needs must be written by the REPL to a named store (files work) and passed to `SPAWN` by name.
- **\[C\] Why RLM binds the model call inside the REPL anyway.** Fan-out cost. Splitting into 1,000 chunks and spawning per chunk under design (a) means the model emits 1,000 tool calls through inference: slow, expensive, error-prone. In code it's a loop: `for ch in chunks: spawn(...)`. That is "inference as the tool of last resort" applied to orchestration: let the model write the loop once, let code execute it. As understood, RLM exposes its recursive model call as a function inside the REPL for this reason.
- **\[C\] Design (b): ISA bindings inside the sandbox.** Not tunneling; the second entry point applied to model-written code. A client library in the sandbox (`spawn`, `infer`, `read`, `query`, `put`) talks to the environment over a channel. Every call is attributed to the owning agent, checked against its capabilities and budget, and journaled, like any other call.
- **\[C\] Does the ISA include sandboxes?** The ISA specifies the **guarantees**, not the mechanism, as a CPU ISA specifies protection semantics while the MMU implements them. Proposed: a standard extension (v0.2's `E`) defining a code-execution session tool, containment guarantees (no access beyond the agent's namespace and caps; resource limits), and optional ISA bindings. The environment chooses the technology.
- **\[C\] Durability.** REPL sessions are ephemeral by nature. Large values go to named stores; each executed cell is journaled; a lost session is re-created and, if its code was side-effect free, rebuilt by replaying cells. Otherwise the agent is told the session was lost.
- **\[C\] Answer.** Design (a) works with v0.3 today, bridged by stores. Design (b) is a v0.4 extension, and the same mechanism covers case T (installed programs) and case X at scale.

### How ISA calls reach the sandbox

- **\[R\]** Now understands sandboxes and why ISA calls are needed inside them. One path: implement the ISA calls as a Python library imported into the REPL; the inference call then needs the library's documentation in context; and the library must actually work, which seems to be part of the sandbox implementation. Implementation detail, but it should be supported; update the documents to cover it.
- **\[C\]** The library is a **thin stub**, not an implementation: each function marshals its call over a channel to the environment, which does the real work as the owning agent. Same shape as a syscall wrapper trapping into the kernel. The agent's identity reaches the environment as a capability token injected when the session starts; the sandbox can't forge or widen it.
- **\[C\]** "Making sure it works" is solved by generation, not hand-coding: one interface definition produces the model's tool schemas, the library stubs, and the documentation, plus conformance tests running each instruction both ways.
- **\[C\]** Documentation cost: the library docs are a stable segment, so they belong in the shared canonical prefix (cached across agents) or are disclosed progressively.
- **\[C\]** Written up as R17–R21 under "v0.4 requirements" above. v0.3 left unchanged.

## Raw notes: Pi extensions and edge devices (Oct 20, early)

Not yet integrated. Cases Y and Z were added to the port test above.

- **\[R\]** Test case: Pi agent extensions. They seem to fall into four to six categories; survey them, note the most popular, and evaluate whether the architecture supports all of them. Never reviewed any personally.
- **\[R\]** A self-adapting agent should choose which models it works with. Test case: the harness configures itself to run on a low-resource edge device with local models, a large delta in capability and workflow from a good laptop using only hosted frontier models.

### Pi extensions survey

- **\[C\] Scale and packaging.** The [Pi package catalog](https://pi.dev/packages) lists about 5,520 npm packages, sortable by monthly downloads. Package types are extension, skill, theme, and prompt template; many packages bundle several. Packages around rank 150–200 draw roughly 6,000–7,600 downloads a month. The very top ranks weren't retrievable, so "popular" below means well-downloaded, not a ranked top ten.
- **\[C\] What an extension can do** ([extension docs](https://pi.dev/docs/latest/extensions)): a TypeScript module that registers tools, slash commands, shortcuts and flags, model providers, MCP servers, virtual models that route each request, terminal renderers and UI, inter-extension events, durable session entries, and session controls (active tools, model, thinking level). Lifecycle events cover input, agent runs, model turns, provider calls, messages, tools, and settlement. Handlers notify, mutate in place, chain transformations, gate (block a tool call), or act at boundaries (`turn_end`, `agent_before_settle` can append entries or request one more continuation). Extensions run in the Pi process with full OS permissions.
- **\[C\] Functional categories** (synthesized from the API and a sample of packages):

| Category | Typical packages | Pi mechanism | HB support |
| --- | --- | --- | --- |
| Tools and integrations | Web search and scraping (pi-web), Linear (64+ tools), computer use, LSP, codebase indexes, MCP bridges | `registerTool`, `registerMcpServer`, tool annotations (read-only, destructive, idempotent, open-world, as in MCP) | Supported: `CALL` and namespace. Pi's annotations match R6 effect classes; adopt MCP's names |
| Providers and model routing | Ollama Cloud, TokenRouter, Cursor SDK, Antigravity bridge, llama.cpp server discovery, per-subagent model routers | `registerProvider`, virtual models, model and thinking controls | Supported by R23 (codecs) and R27 (roles, fallbacks); a virtual model is a role resolved by a routing program |
| Context, memory, compaction | Iterative auto-compaction, observational memory with cache-friendly tiers, "Codex-style" context windows, state and memory compilers, prompt-cache tuning, context firewalls | `context` and `context_with_system` (transform the transcript per request), compaction entries, `cache_warming_decision`, `appendEntry` | Supported, and a strong match: Pi's per-request `context` transform is our composer; `cache_warming_decision` is R16 keepalive; `appendEntry` is a journal note |
| Orchestration: subagents, teams, workflows | Async subagent fleets with nested delegation, tiered parallel teams with file locks, crews with worktrees, workflow graphs with recovery, Ralph loops | Tools that run nested work; `ctx.executeTool` with parent ids; continuation at settle | Supported: `SPAWN`, receipts, journal. File locks and peer messaging need the sketched IPC and leases |
| Safety, policy, guardrails | Permission gates, safe-guard and git-guard, rule wardens judging every write, secret redaction, container sandboxes, plan mode that removes editing tools | `tool_call` gate, `tool_result` redaction, `input` transform, `project_trust` before loading project resources, active-tool changes | Supported via `ON` handlers, R25 verdicts, R17 sandboxes. Plan mode exposes a gap: changing an agent's namespace mid-run |
| Observability | OpenTelemetry span trees, telemetry, token-speed meters, notifications, status lines | Lifecycle events, `provider_stream_event` | Supported via events and the journal (R26) |
| UI and interaction | Themes, status lines, renderers, editors, voice I/O, slash commands | `ctx.ui`, renderers, shortcuts, commands | Out of scope by design: the user is a device behind the `user` tool. Slash commands are user-triggered programs |
| Process and workflow content | Spec-driven frameworks, dev workflows, commit and review conventions, presets | Skills and prompt templates | Content, not mechanism: skills and prompts in plans |

- **\[C\] Gaps the survey exposes:**
  1. **Namespace changes mid-run.** Pi changes active tools during a session (plan mode, dynamic loading, late MCP connections, `tool_search`). v0.3 fixes a namespace at `SPAWN`. Needed: a journaled namespace change, with its cost stated, since Pi notes tool changes can invalidate the cached prefix.
  2. **Provider-payload hooks.** Pi lets extensions mutate provider headers and serialized requests and observe raw stream events. In HB that's below the line: it belongs in codec plug-ins (R23), not the ISA, so portable programs never depend on one provider's wire format. Deliberate exclusion, but it should be stated.
  3. **Inter-extension events.** Pi's `pi.events` bus is IPC between programs; covered by the sketched channels.
- **\[C\] Precedents worth borrowing:** Pi's *codemode* tool exposure lets scripts call other tools through `ctx.executeTool`, with nested calls carrying `parentToolCallId` and their token usage rolled up into the caller. That is R18 (ISA bindings in a sandbox) and R26 (provenance) already in production. Pi's `tool_search` with deferred exposure is progressive disclosure. Its `agent_before_settle` continuation is our HALT handler (case B).
- **\[C\] Verdict:** every functional category maps onto HB except UI (out of scope by design) and provider-payload hooks (below the line by design). One real gap: mid-run namespace changes.

### Model selection and edge-device self-configuration

- **\[C\]** This turns model roles (R27) from configuration into a decision the agent makes. Requirements beyond R27:
  1. **Environment description in `CAPS`.** Not just models but the machine: memory, accelerators, storage, network availability, and local inference servers with their concurrency limits. A local server usually runs one model at a time with limited KV memory, so parallel fan-out that helps on hosted APIs hurts here.
  2. **Model discovery and characterization.** Find available models (Pi packages already scan for llama.cpp, LM Studio, vLLM, and similar servers), then measure them on the agent's own tasks (latency, tokens per second, success rate) and record results in a store for future choices.
  3. **A cost model that changes units.** On hosted models tokens cost money; locally they cost time and energy, and caching becomes the local server's prefix cache. Budgets need time and energy as first-class units, and the token metric becomes "tokens per unit of the scarce resource."
  4. **Workflow selection, not just model selection.** A small model with a short context calls for smaller views, more deterministic code, stricter decoding (`require_call`, constrained output), shallower recursion, and fewer agents. The self-adapting agent chooses a composer policy and an orchestration style along with the models.
- **\[C\]** CPU parallel: RISC-V has an **embedded base variant (RV32E)** with half the registers, plus separate profile families for application processors (RVA) and embedded-class systems (RVB). A small-context local model is the E variant of this ISA: same instructions, fewer registers. An HB profile for embedded targets (say "HE-embedded") would define the guaranteed minimum.
- **\[C\]** Proposed v0.4 candidates: R29 environment description in `CAPS`; R30 model discovery and characterization records; R31 time and energy budgets; R32 an embedded-class profile; R33 journaled mid-run namespace changes.

### 12-Factor Agents review (Oct 20, morning)

- **\[R\]** Reviewed [12-Factor Agents](https://github.com/humanlayer/12-factor-agents); the architecture seems to support nearly everything in scope. Asked for Claude's view.
- **\[C\]** Agree. Mapping, from the README and the factor pages for 3, 5, 8, 12, and 13:

| Factor | HB counterpart | Fit |
| --- | --- | --- |
| 1 Natural language to tool calls | Everything is a tool call | Clean |
| 2 Own your prompts | Plans and missions are caller-owned data | Clean |
| 3 Own your context window (custom formats, e.g. all events packed into one message; hide resolved errors) | Composer and `events(selector, render)`; chat roles stay below the line in codecs; liveness hints | Clean |
| 4 Tools are just structured outputs | Decode → hooks → dispatch: a call is an intent that code may execute, change, or refuse | Clean |
| 5 Unify execution and business state | The journal: execution state is derived from it; serialize, recover, and fork for free | Clean, near-identical rationale |
| 6 Launch, pause, resume with simple APIs | `SPAWN`, `SIGNAL`, durable receipts | Clean |
| 7 Contact humans with tool calls | The `user` tool, extended to all output | Clean, we go further |
| 8 Own your control flow (pause between tool *selection* and *invocation*) | Write-ahead journaling of the call before dispatch plus an `ON(CALL, Enter)` approval handler: a crash in between is safe | Clean, and the write-ahead rule is exactly what makes it durable |
| 9 Compact errors into the context window | Error results, repair tiers, liveness hints to drop resolved errors | Clean |
| 10 Small, focused agents | Subagents with small composed views | Clean |
| 11 Trigger from anywhere, meet users where they are | The `user` tool as a device that can be Slack or email | **Partial**: no defined way for an external event (webhook, cron, email) to start or wake an agent |
| 12 Stateless reducer (agent as a fold over events) | Already adopted: state is a fold over the journal | Clean |
| 13 Pre-fetch context you'll likely need | A composer calls tools deterministically before `INFER`; "inference as the tool of last resort" | Clean |

- **\[C\] Two gaps:**
  1. **External triggers.** The HEI should define ingress for outside events (webhooks, schedules, messages) that spawn or wake an agent, passing through the ingress guardrails (R25) like any other content.
  2. **Durable sleep and wait-for-event** (named in factor 8). No new instruction needed: a `timer` or `wait_for` tool whose receipt completes at a time or on an event, surviving restart like any receipt.
- **\[C\] One tension worth keeping in view.** 12-Factor is anti-framework: the caller owns the loop (it links to "library vs. framework" arguments). HB's workflow mode is library-shaped; agent mode, where the environment drives the loop, is framework-shaped (inversion of control). Supporting both is right, but agent mode must stay thin and fully hookable, so nobody has to "reverse-engineer the framework" to get past 80% quality, which is the failure 12-Factor was written about.
- **\[C\] Where HB goes beyond 12-Factor:** multi-agent IPC, cache economics, model selection and edge targets, sandboxes with bindings, provenance, and the portability goal itself. 12-Factor is a methodology for one team's agent; HB aims to be the substrate such agents are written on.
- **\[C\]** Proposed v0.4 candidates: R34 external triggers in the HEI; R35 durable timer and wait-for-event tools.

### BabyAGI 2 and self-building agents (Oct 20)

- **\[R\]** Shared a post by Yohei Nakajima (x.com/yoheinakajima/status/1840678823681282228).
- **\[C\]** X blocks automated reading, so the post itself wasn't visible. Its id dates it to about Sept 30, 2024, which matches the launch of **BabyAGI 2**, Nakajima's rebuild of BabyAGI as "an experimental framework for a self-building autonomous agent"; treat that identification as likely, not confirmed. Sources: [BabyAGI repo summary](https://www.sourcepulse.org/projects/1262470), [BabyAGI 2o](https://www.sourcepulse.org/projects/1830193), [AI Tinkerers talk notes](https://one-shot.aitinkerers.org/p/how-baby-agi-2-reimagines-ai-s-ability-to-build-its-own-tools).
- **\[C\] What it is.** Core is **functionz**: functions stored in a database rather than code files, with a graph of imports, dependencies between functions, and secrets; automatic loading; a log of every execution; triggers that run functions on events; a web dashboard. BabyAGI 2o (Oct 2024) is the minimal companion: an agent that writes Python functions as tools, registers them, installs packages they need, and iterates on errors. Nakajima's framing: "What is the simplest thing I can build that can build itself?" His levels of tool autonomy: 0 human-crafted tools only, 1 tools created on request, 2 tools created when the agent sees a need, 3 tools created in anticipation of need.
- **\[C\]** This is case T and R20 (installed programs) as a working system, and the closest prior art to the self-adapting agent. Mapping:

| BabyAGI 2 / functionz | HB | Fit |
| --- | --- | --- |
| Function database with metadata | Installed programs in a store, registered in namespaces (R20); `QUERY` to search them | Clean |
| Dependency graph (imports, other functions) | Program manifest: dependencies and requested capabilities | Clean, with capabilities added |
| Execution log for every call | Journal with provenance (R26) | Clean |
| Triggers on events | `ON` handlers; external triggers (R34) | Clean |
| Agent writes and registers its own tools (2o) | Sandboxed code sessions (R17), ISA bindings (R18), installation (R20) | Clean in design; v0.3 can't yet |
| Auto-installs packages | Must run inside the sandbox under network and package policy | HB stricter, deliberately |
| Functions run in-process with full permissions | Programs run sandboxed with declared capabilities | HB stricter, deliberately |
| Key wrapper storing API keys | **Gap:** no secrets model in HB | Gap |

- **\[C\] New gap: secrets.** Generated programs need credentials, but models must never see them and the journal must never record them. Proposed: secrets are capabilities, not values. A program declares the secret names it needs; the environment injects them inside the sandbox or tool at call time; journal and views record only the reference.
- **\[C\] Lessons for case T.** Use Nakajima's levels as the acceptance ladder: level 1 (install on request) first, then 2 (on need), then 3 (anticipatory). Add a **verification gate** before registration: a generated program must pass tests in its sandbox before it enters any namespace, and every registration is versioned and revertible. BabyAGI 2o's error-iteration loop is the inner loop; the gate is what makes it safe to keep the results.
- **\[C\]** Proposed v0.4 candidate: R36 secrets as capabilities. Case T gains the level ladder and verification gate.

### Inversion of control: model-written code promoted into the harness (Oct 20, afternoon)

- **\[R\]** Tools are meant for the model to call; the ISA is meant for harness code to call: opposite directions. Sandboxes let the model call the ISA too, and RLM adds code written by the model and called by the model. Given the goal of self-modifying harnesses (give it a task such as customer service and let it optimize itself), there is a continuum: a sandbox script used once; a sandbox function called often; and harness code called by other harness code, not by the model. With inference as the last resort, we want the model to develop code that gets **integrated into the harness**: the model writing code for the harness to call, rather than the harness providing tools for the model to call. That feels like an inversion of control.
- **\[C\]** The continuum, by author and caller:

| Stage | Written by | Called by | Example |
| --- | --- | --- | --- |
| Harness tool | Human | Model | `read`, `bash` |
| ISA from harness code | Human | Harness code | A composer calling `QUERY` |
| ISA from the model | Human (bindings) | Model | `spawn` in a sandbox |
| Scratch script | Model | Model, once | RLM chunking code |
| Installed tool | Model | Model, repeatedly | BabyAGI 2o function, case T |
| **Promoted harness code** | Model | Harness code; the model is no longer in that path | A hook, composer, guardrail, or workflow step that replaces an inference step |

- **\[C\]** The last row is the inversion: the model becomes the harness's developer, and the harness becomes the runtime for the model's code. Classic inversion of control is "the framework calls your code"; here, the model *writes* the code the framework calls.
- **\[C\] Closest precedent: JIT compilation.** An interpreter (the model) executes; profiling finds a hot path; the JIT compiles it to native code (deterministic harness code); afterwards the interpreter isn't involved on that path. JITs also solve the correctness problem: compiled code carries **guards** for its assumptions, and when a guard fails it **deoptimizes** back to the interpreter. Promoted code should do the same: declare when it applies, handle the common case deterministically, and fall back to inference when the guard fails. That is "make the common case fast" and "inference as the last resort" in one mechanism.
- **\[C\] Shell precedent for the lifecycle:** a typed one-liner → a saved script → a command on `PATH` → a cron job or daemon. Same code, rising permanence and trust.
- **\[C\] Why HB makes promotion cheap:** every stage uses the same ISA through the same generated interface (R18, R19). A program doesn't care who wrote it or who calls it, so promotion is a change of **registration and privilege**, not a rewrite.
- **\[C\] The danger is privilege escalation.** Code in harness position (hooks, composers, guardrails) can rewrite data flows and gate other agents; a tool can only answer when called. Promoting into the harness is like loading code into the kernel. Precedent: **eBPF**, where user-supplied programs run inside kernel hooks only after a verifier accepts them (Pie cites eBPF in its lineage). Promotion into harness positions needs a stronger gate than installation as a tool: tests, evidence, and human approval by default.
- **\[C\] The agent being optimized must not be able to rewrite its own guardrails or authority rules.** Self-modification is limited to its own level; changes to policy-level code go through a separate, higher-privilege path. Same structure as the self-improving agent experiment (git for versioning and rollback, a watchdog as the external check).
- **\[C\] Evidence for promotion comes from the journal**, the JIT's profiler: how often a sequence recurs, its tokens per occurrence, its success rate. A promotion decision can be data-driven ("this lookup pattern appears in 40% of tickets at 3,000 tokens each").
- **\[C\] Proposed v0.4 candidates:**
  - **R37 Promotion lifecycle.** Stages: scratch → saved → installed tool → harness program (hook, composer, guardrail, workflow step). Each promotion is gated (tests, journal evidence, review), versioned, revertible, and journaled with authorship.
  - **R38 Guarded replacement with inference fallback.** Promoted code declares a guard; on a miss, control falls back to the model (deoptimization). Fallback rates are tracked, so code that misses too often is demoted.
  - **R39 Privilege levels for installed code.** At least: tool level (called on request) and harness level (hooks, composers, guardrails), with policy-level code outside any agent's own self-modification rights. Connects to the open privilege question in the CPU reference.

### What "self" means; JIT as the frame (Oct 20, afternoon)

- **\[R\]** In the self-improving agent experiment, "self" was literal: the script handed itself to a model and overwrote its own file. For this project "self" should be looser. A tiny-model edge device lacks the competence to modify itself. What's really meant is one harness acting on a target harness: an **authoring harness** (generous hardware, frontier models) builds a deployable **target harness**, either forked from itself as a template and then reconfigured, or built from nothing but the ISA implementation.
- **\[R\]** At that point the authoring harness is a coding agent whose application happens to be an agent, or, given the agent/workflow continuum, an application built on the ISA that uses language models and follows harness design principles. The authoring agent fits that pattern itself: its use case is building agentic applications.
- **\[R\]** True self-modification remains as one configuration: iterations where the target is the authoring agent's own code, improving it at building agents.
- **\[R\]** The JIT analogy fits model-authored code promoted into harness execution; explore JIT principles for the project. They may not change the ISA much but should shape the harness built on top, starting with the authoring agent.
- **\[C\]** Written up as a new document, **Reference: JIT Compilation Principles**: the authoring/target reframing; JIT vs. AOT (edge targets are AOT with profile-guided optimization); self-hosting and bootstrapping as the self-improvement case; a catalog of JIT principles (tiered compilation, type feedback, inline caches as decision caches, guards and deoptimization, on-stack replacement, tracing, partial evaluation and the Futamura projections, invalidation); ten mechanisms for the authoring agent; risks including Thompson's "trusting trust"; and ISA impact.
- **\[C\]** New v0.4 candidates from it: R40 declared assumptions and invalidation; R41 sampled audit routing; and R38 strengthened so a guard miss is a standard environment-handled outcome.

* **\[R\] Correction to the JIT framing:** these are AI applications; their value is tasks only a model can do, so some inference calls are irreducible. Authoring code isn't justified only by eliminating a call. For irreducible calls, the JIT goal is making them cheaper: call less often (architecture, algorithm) and use fewer tokens per call (more concise and accurate data, shorter and more effective instructions, more output structure, better tools). Applies on the edge too: a voice-controlled smart thermostat on Cactus's Whistle → Needle stack will never drop its models.
* **\[C\]** Agreed and applied to the JIT reference: two goals (move what can be code into code; make remaining inference cheaper), plus a new section mapping compiler optimizations onto irreducible inference: loop-invariant motion (ask once per task), memoization, batching as vectorization, inlining adjacent model steps, prompt specialization as constant folding, composition as register allocation, cheaper roles as strength reduction, constrained output as narrower types, better tools as a richer instruction set, and fine-tuned small models as custom instructions. Levers: frequency, tokens per call, quality per call; all measurable from the journal.

### From prototype to deterministic software (Oct 20, afternoon)

- **\[R\]** Since JIT principles can eliminate inference calls entirely, the system can also author non-agentic software: if every call is eliminated, the result is a fully deterministic application. Ordinary development becomes bootstrapping an ill-defined agentic application, where anything not yet in code is performed by the model, given tools that connect its output to whatever code exists so far. A path from prototype to production.
- **\[R\]** Drivers are metrics: built-in performance metrics (inference calls, tokens, wall time, hardware use) and, most important, whether the app does what the developer or user needs. Two ways to measure that: a spec up front, with test cases derived from it and pass rate as the metric (TDD, somewhat waterfall); or old-fashioned prototyping, where the developer shares an idea, the authoring agent iterates, the developer plays with the result and gives feedback.
- **\[R\]** This separates cleanly: the authoring agent's harness isn't touched, so it isn't self-modification; it's an authoring agent developing a target application.
- **\[R\]** Related idea: bake AI into any product so it can iterate on itself (message your messaging app "I'd like this feature" and it builds it). Really two entities, a coding agent and a target app, that may be conveniently bundled. Per-user customization of installed apps (a personal instance on one phone) needs source, builds, and installs; it is a separate project. **Not a requirement for this project.**
- **\[C\]** Precedents for the methodology:
  - **Strangler fig pattern** (Martin Fowler): replace a system piece by piece while it keeps running. Here the model is the system being strangled, and code grows around it.
  - **Wizard of Oz prototyping**: a hidden human performs the behavior until it's built. The model is now the wizard.
  - **"Make it work, make it right, make it fast"** (Kent Beck): the model makes it work, tests make it right, JIT makes it fast.
- **\[C\] Feedback becomes tests, through the journal.** In the prototyping path, every model-driven run the developer approves is recorded in full. Those approved traces are an executable spec: when code replaces a model step, it must reproduce the approved behavior on the recorded inputs. That is **characterization or golden-master testing** (Michael Feathers' technique for legacy code), with the model's approved outputs as the oracle. It gives JIT verification a concrete test suite without a written spec, and the two drivers combine: spec-derived tests plus approval-derived tests.
- **\[C\] The limit case: eject the runtime.** When the last inference call is gone, the application may not need the harness at all. The authoring agent should be able to emit plain code with no ISA dependency, like an "eject" step, optionally keeping the journal for observability. Dead-code elimination of the harness itself.
- **\[C\]** Scope: no ISA changes needed; this is an authoring-agent capability and a methodology. Per-user app customization is parked as out of scope; prior thinking exists under the name "malleable software" (as recalled, an Ink & Switch essay), if it's ever revisited.

### Reference library expansion (Oct 20, afternoon)

- **\[R\]** Asked for POSIX principles, general compiler principles (including whether an IR analog fits), and a survey of all of computing (networking, databases, and more) for established principles; Claude to outline areas, dip into each, and add references as fit.
- **\[C\]** Added three documents: **Reference: Survey of Computing Principles** (map of 18 domains, six strongest imports, priority list for further deep references), **Reference: POSIX and Operating Systems**, and **Reference: Compiler Principles**.
- **\[C\] IR answer:** the harness has three IRs. The plan is the main one, lowered plan → view → provider request with the codec as the back end; the journal is the canonical model-neutral IR of history; and a new trace IR feeds the JIT. The third back-end concern from early on was instruction selection.
- **\[C\] Strongest new imports from the survey:** prompt injection as a confused deputy, fixed by capability scoping (quarantined reader pattern); tool calls as algebraic effects with handlers; the end-to-end argument (the ISA provides mechanisms, applications own correctness); composition as cost-based query planning with summaries as materialized views; trap-and-emulate for nested harnesses; hysteresis against thrashing.
- **\[C\] New v0.4 candidates:** from POSIX, subtree signaling, "inheritance only narrows," conventions as part of the spec, no ambient authority; from compilers, R42 plan IR with verifier, R43 shared composition passes, R44 codecs as back ends, R45 trace IR, R46 declared resources for independence analysis.
- **\[C\] Recommended next deep reference:** security principles, before specifying authority labels, guardrails, privilege levels, secrets, and promotion gates.

### Security and networking references (Oct 20, afternoon)

- **\[R\]** Sockets fit the earlier pipes/RPC/IPC discussion of agent-to-agent communication; consult networking principles, first for agent-to-agent communication. Go ahead with security research, and dig into any other especially relevant area.
- **\[C\]** Added **Reference: Security Principles** and **Reference: Networking and Agent Communication**.
- **\[C\] Security headlines:** structure over detection (don't make the deputy read more carefully; give it less to be confused with); Saltzer and Schroeder's ten principles mapped to harnesses; object capabilities with attenuation, revocation, membranes; agent-specific defenses (lethal trifecta, Meta's Agents Rule of Two, dual-LLM, CaMeL's control/data-flow separation with capability tags, FIDES information-flow labels). v0.4 candidates: R47 confidentiality and integrity labels with deterministic propagation; R48 quarantined inference mode; R49 Rule-of-Two check at spawn and across channels; R50 attenuation, revocation, membranes.
- **\[C\] Networking headlines:** a subagent is a *remote* call in Waldo's sense even in-process (latency, partial failure, concurrency, no shared memory); sockets as the model for one channel abstraction over many transports; MCP is the agent-to-tool layer, A2A (which absorbed IBM's ACP in Aug 2025) the agent-to-agent layer, ANP decentralized peers; they are transports, not ISAs. v0.4 candidates: R51 channel types over pluggable transports; R52 remote agents via A2A (Agent Card as `CAPS`, task as receipt, push as interrupt) and exposing HB agents as A2A servers; R53 bounded mailboxes and backpressure; R54 heartbeats and liveness. Plus idempotent messaging, end-to-end acknowledgment, peers untrusted by default.
- **\[C\]** Observation: the design has converged on an actor-style, capability-secured messaging system with durable logs, a well-studied family (Erlang/OTP, E, Akka, log-based systems).
