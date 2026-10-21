# Harness ISA — Draft v0.1

Oct 19, 2026 · @Rene Ravenel

## Purpose and scope

This draft defines a small, universal set of primitives that any LLM harness could expose, so agent programs and skills become portable across harnesses the way C programs are portable across POSIX systems.

The framing: the model is the CPU, tokens in and tokens out are its native instruction stream, and a tool call is the **trap** instruction. When the model emits a call, control passes to the harness (the kernel), which services it and resumes the model. The Harness ISA is therefore one interface, not two stacked ones: it is the harness's syscall table.

In scope: the primitives, their semantics, how both models and deterministic code reach them, and a portable baseline versus optional extensions. Out of scope: model training, wire protocols, and UI.

## Design principles

1. **One primitive set, two entry points.** Every primitive is reachable both as a tool the model can call (nondeterministic path) and as a library call from code running inside the harness (deterministic path). Same names, same semantics.
2. **Small core, derived everything else.** Sub-agents, loops, map-reduce, and retries are compositions, not primitives. Spawn is fork plus an infer loop, as a Unix process is `fork` plus `exec`.
3. **Portable baseline, optional extensions.** The core must be implementable on top of any hosted chat API at the message level. Token- and KV-level control is an extension a capable engine may advertise, in the spirit of `ioctl`.
4. **Capabilities, not trust.** Every call runs under a capability set (which tools, stores, budgets, and children it may touch). Skills and sub-agents request capabilities; the harness grants or denies.
5. **Everything observable.** Every primitive emits a lifecycle event. Hooks, logging, budgets, and policy are built on events, not special cases.
6. **Explicit cost.** Every call returns what it consumed (tokens, latency, money), so schedulers and optimizers can reason about it.

## Machine model

The harness is a kernel with two kinds of callers and four kinds of resources, joined by one capability-checked instruction set.

&#91;embedded content: machine model · two callers, six instruction groups, four resources\]

A **context** is the unit of execution, the analog of a thread: an ordered message history with a capability set and a budget. Forking a context is how parallelism and sub-agents arise. The model never touches resources directly; it only traps, and every resource is reached through an instruction.

## Core instruction set

Twenty instructions in six groups make up the portable baseline. Handles (`ctx`, `store`, `tool`, `sub`) are opaque and capability-checked. Every call also returns a cost record.

| Group | Instruction | Signature | Semantics |
| --- | --- | --- | --- |
| Context | `CTX.NEW` | `(system?, caps) -> ctx` | Create an empty context with a capability set. |
| Context | `CTX.APPEND` | `(ctx, role, content) -> ctx` | Add a message, tool result, or attachment. |
| Context | `CTX.FORK` | `(ctx) -> ctx'` | Copy-on-write branch sharing the prefix. Cheap where the engine shares prefixes. |
| Context | `CTX.SLICE` | `(ctx, range) -> ctx'` | Truncate or select messages, e.g. drop a failed branch. |
| Context | `CTX.COMPACT` | `(ctx, policy) -> ctx'` | Lossy shrink: summarize or evict per policy. Emits what was dropped. |
| Context | `CTX.SNAPSHOT` | `(ctx) -> ref` / `(ref) -> ctx` | Persist and restore a context. |
| Infer | `INFER` | `(ctx, model, constraints) -> stream` | The one compute instruction. Constraints: schema, stop, max tokens, tool set, budget. |
| Infer | `DECODE` | `(stream) -> {text, calls[], done}` | Classify output: text for the user, tool calls to dispatch, or end of turn. |
| Tool | `TOOL.LIST` | `(caps) -> specs[]` | Discover tools available under current capabilities. |
| Tool | `TOOL.CALL` | `(tool, args) -> result` | Invoke an external function. Synchronous or async. |
| Tool | `TOOL.AWAIT` | `(handles[]) -> results[]` | Join one or more async handles: tool calls or inference streams. |
| Store | `STORE.GET` | `(store, key) -> value` | Read persistent memory. |
| Store | `STORE.PUT` | `(store, key, value)` | Write persistent memory. |
| Store | `STORE.QUERY` | `(store, query, k) -> hits[]` | Search: keyword, vector, or hybrid. RAG lives here. |
| Human | `HUMAN.EMIT` | `(content)` | Show content to the user. |
| Human | `HUMAN.ASK` | `(prompt, options?) -> reply` | Block for user input, approval, or choice. |
| Event | `EVT.ON` | `(event, handler) -> sub` | Subscribe to a lifecycle point. |
| Event | `EVT.OFF` | `(sub)` | Unsubscribe. |
| Event | `EVT.SIGNAL` | `(target, signal)` | Send cancel, pause, resume, or a budget warning to a running loop. |
| Meta | `CAPS` | `() -> caps` | Introspect current capabilities and budget remaining. |

Standard events: `pre_infer`, `post_infer`, `pre_tool`, `post_tool`, `pre_compact`, `human_wait`, `budget_low`, `error`, `done`. A handler on a `pre_*` event may allow, deny, or rewrite the call, which is how policy and guardrails attach.

## Derived operations

The familiar harness behaviors are short programs over the core, which is the test that the core is complete.

**The agent loop.** The canonical turn is a loop of infer, decode, dispatch:

```
loop:
  s = INFER(ctx, model, constraints)
  out = DECODE(s)
  if out.text: HUMAN.EMIT(out.text)
  if out.done: return
  results = TOOL.AWAIT([TOOL.CALL(c) for c in out.calls])
  ctx = CTX.APPEND(ctx, "tool", results)
  if over_budget(ctx): ctx = CTX.COMPACT(ctx, policy)
```

**Sub-agent (spawn and join).** `sub = CTX.FORK(ctx)` or `CTX.NEW(task, narrower_caps)`, run the loop on it, then `CTX.APPEND(ctx, "tool", summary(sub))`. Fork when the child needs the parent's history; new when isolation matters more.

**Parallel exploration.** Fork N branches, run each, score, keep the best, `CTX.SLICE` the rest away. Tree-of-thought and best-of-N are this pattern.

**Map-reduce.** For each chunk, `CTX.NEW` plus one `INFER`; reduce with a final `INFER` over the results.

**RAG.** `STORE.QUERY` then `CTX.APPEND` the hits before `INFER`.

**Approval gate.** `EVT.ON(pre_tool, h)` where `h` calls `HUMAN.ASK` for risky tools and denies on no.

**Long-running memory.** On `pre_compact`, `STORE.PUT` what is about to be evicted; on a new session, `STORE.QUERY` to rehydrate.

## Extensions

Extensions are optional instruction groups an engine advertises through `CAPS`. Code checks for them and falls back to the baseline, as POSIX programs probe for optional features.

| Extension | Instructions | What it adds | Baseline fallback |
| --- | --- | --- | --- |
| `X.KV` | `KV.PIN`, `KV.EVICT`, `KV.SHARE`, `KV.PAGE` | Direct KV-cache control: pin a prefix, share pages across forks, evict on demand | `CTX.FORK` as message copy; rely on implicit prefix caching |
| `X.TOKEN` | `FORWARD`, `SAMPLE`, `LOGITS` | Single forward pass, custom sampling, logit access and masking | Whole-turn `INFER` only |
| `X.CONSTRAIN` | `INFER` with `grammar` | Grammar- or regex-constrained decoding | JSON schema constraint, validate and retry |
| `X.SPEC` | `INFER.SPECULATE` | Start tools or child inferences while the parent is still streaming | Sequential infer, then dispatch |
| `X.SANDBOX` | `EXEC` | Run code in an isolated environment with resource limits | A `bash` or `python` tool under caps |
| `X.ADAPT` | `ADAPTER.LOAD` | Swap LoRA adapters or steering vectors per call | Prompt-level conditioning |

`X.KV` and `X.TOKEN` assume the harness runs next to the model, as in a programmable inference engine. Hosted APIs will rarely offer them, which is why they stay out of the core.

## Skill binding

Today a skill can only ask the model to do things. Under this ISA a skill can also carry code that calls the harness directly, so its control flow is deterministic where it needs to be.

A skill package gains two optional parts beside its instructions:

- **Manifest.** Declares the capabilities the skill requests (tools, stores, max sub-agents, token budget) and which ISA version and extensions it needs.
- **Programs.** Entry points written against the ISA library, run by the harness in a sandbox under the granted capabilities. The model can invoke a program by name, like a tool, or a program can run on an event.

Example: a code-review skill whose program forks one context per changed file, runs a fixed review prompt on each in parallel, gathers the results, and only then hands a single summary to the model. No prompt has to persuade the model to fan out; the program does it.

```
def review(ctx, files):
    subs = [CTX.APPEND(CTX.FORK(ctx), "user", REVIEW_PROMPT + f) for f in files]
    outs = TOOL.AWAIT([INFER(s, "fast", {"schema": Finding}) for s in subs])
    return merge(outs)
```

The split mirrors compiled code versus interpreted script: put fixed structure in programs and judgment in prompts.

## Backend concerns

A fixed ISA lets the harness act like a compiler backend, optimizing how a high-level agent program maps onto these instructions. The three classic backend passes carry over directly.

| Compiler pass | Harness analog | Example decisions |
| --- | --- | --- |
| Instruction selection | Choosing how each step runs | Tool vs. small model vs. large model vs. plain code; fusing several steps into one `INFER` |
| Register allocation | Context allocation | What stays in the window, what spills to `STORE`, when to `CTX.COMPACT`, which prefix to keep stable for cache hits |
| Instruction scheduling | Call orchestration | Parallel `TOOL.CALL`, prefetching `STORE.QUERY` early, speculative children, batching sibling forks |

Two differences from a CPU backend shape the design. Liveness is uncertain, since the harness cannot know whether a detail will matter later, so eviction is a prediction rather than a proof. And `CTX.COMPACT` is lossy, so unlike register spilling it can change behavior; the ISA requires it to emit what it dropped so later passes can recover.

## Open questions

- [ ] **Is `DECODE` a primitive or a detail of `INFER`?** Separating it lets code inspect output before dispatch, but most engines fuse the two.
- [ ] **Where does a sub-agent's identity live?** A forked context alone has no address. Signals and capability narrowing may need a first-class `sub` handle after all.
- [ ] **Compaction semantics.** What must `CTX.COMPACT` guarantee: a summary, a pointer to evicted content in `STORE`, or both?
- [ ] **Determinism budget.** How should a program express "retry until valid" without unbounded cost? Likely a standard budget field on every call.
- [ ] **Model heterogeneity.** Can one `INFER` contract cover models with different tool formats, reasoning modes, and context limits, or does selection belong in the caller?
- [ ] **Security of programs.** Skill programs run with real capabilities. What is the minimum sandbox and review model?
- [ ] **Versioning.** How the ISA evolves without breaking skills, given that hosted APIs change faster than POSIX ever did.
- [ ] **Reference implementation.** Which existing harness is the cheapest place to prototype: a minimal one such as Pi, or a programmable engine?
