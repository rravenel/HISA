# Harness ISA — Draft v0.3

Oct 19, 2026 · @Rene Ravenel

v0.3 rebuilds the design around a durable journal and composed views, and shrinks the base from 18 instructions to 13. It is written to be the spec the first executable test is built against: the core is specified precisely, and everything else is deliberately sketched.

## What changed since v0.2

| Area | v0.2 | v0.3 | Source |
| --- | --- | --- | --- |
| Source of truth | The context | A write-ahead **journal**; contexts are composed **views** | Loops session; event sourcing |
| Context instructions | `CTX.NEW`, `APPEND`, `READ`, `FORK`, `SLICE` | Gone. Programs hold **plans** as plain data; `COMPOSE` turns a plan into a view | R12 |
| Agent loop | `RUN` instruction | The environment's loop, started by `SPAWN`; ended by `return` or a call to the `user` tool | R2 |
| User I/O | `HUMAN.EMIT`, `HUMAN.ASK` | The user is a tool | Everything is a tool call |
| Calls | Synchronous by default | **Asynchronous by default**; every call returns a receipt; blocking is `AWAIT` | R14 |
| Interrupts | Not defined | Delivered at instruction boundaries; coalesced | R14 |
| Decode | Hidden in `INFER` | `rejects[]`, illegal-call trap, repair policy | R9 |
| Tool results | Unspecified | Standard result shape with status and declared effect | R1, R6 |
| Stores | `STORE.GET`, `PUT`, `QUERY` | Generic `READ`, `PUT`, `QUERY` over a namespace that includes the journal | R15 |
| Durability | Extension `P` | Base property: recovery is a replay of the journal | R5, R11 |

**Fully specified in v0.3:** the journal, `COMPOSE`, `INFER` and decode, tool result shape, async calls with receipts, interrupts, `SPAWN` and the agent loop.

**Sketched only:** IPC channels, namespace mounts, authority propagation, effect classes beyond declaration, cache hints. These have placeholders in the core (fields, names) so the first build doesn't foreclose them.

## Machine model

One **environment** runs one event loop over many **agents**. Each turn of an agent goes around the journal: events are collected, a composer builds a plan, `COMPOSE` turns it into a view, `INFER` runs on it, decoding yields calls, calls are dispatched asynchronously, and their completions come back as interrupts for the next turn.

&#91;embedded content: one turn of an agent · eight steps around the journal\]

Three properties follow from the picture. The journal is written before every effect, so any turn can be replayed. Views are rebuilt from the journal, so no context is precious and any agent can be resumed, re-pointed at another model, or recomposed smaller. And `INFER` is the only atomic step: everything that arrives while it runs waits for the next boundary.

The environment interface (HEI) carries over from v0.2: it defines initial state, the models and tools available, trap handling, budgets, and preemption. v0.3 adds the journal and the scheduler's coalescing window to it.

## The journal

The journal is an append-only, durable, local log of every event in a system of agents. It is architectural state, like memory: every instruction writes to it as a side effect, and nothing else is authoritative.

**Event schema.** Every entry has:

| Field | Meaning |
| --- | --- |
| `eid` | Monotonic id, unique within the journal |
| `ts` | Wall-clock time |
| `agent` | Name of the agent (or program) that caused it |
| `kind` | One of the kinds below |
| `authority` | Authority level of the content (sketched; see Sketched areas) |
| `payload` | Kind-specific content, stored in full |
| `refs` | Related eids: the call a result answers, the view an inference used |
| `cost` | Tokens (input, cached input, output) where applicable |

**Kinds.** `infer.request` (the composed view, by plan and content hash), `infer.result` (raw output, parsed calls, rejects), `call` (tool, args, mode), `result` (tool result), `interrupt` (delivered to an agent), `signal`, `spawn`, `return`, `note` (written by a program), `trap`, `opaque` (provider blobs that must be replayed verbatim; marked with the model that produced them).

**Write-ahead rule.** An event is committed before it takes effect. A call is journaled before dispatch; a result is journaled before anyone can read it; a view is journaled before inference runs on it. If the process dies at any point, the journal says exactly what was and wasn't done.

**Recovery.** On restart, the environment replays the journal. Agents resume from their last committed events. A call with no recorded result is re-issued if its tool is declared idempotent, otherwise surfaced to its agent as a result with status `unknown`. Programs keep no state outside the journal and stores, so a program resumes by re-running from the top and reading where it was (the stateless-reducer rule).

**Retention.** Out of scope for v0.3, but the journal must support redaction of a payload while keeping its event, so secrets can be removed without breaking history.

## Core instruction set

Thirteen instructions in five groups. Data instructions use a small set of generic verbs over a **namespace**, so the journal, stores, and (later) channels are all addressed the same way. Every instruction that can take time returns a **receipt**; every instruction is reachable both as a model tool call and as a library call.

| Group | Instruction | Signature | Semantics |
| --- | --- | --- | --- |
| Data | `READ` | `(name, key \| range) -> value \| events` | Read a store entry or a range of journal events |
| Data | `PUT` | `(name, key, value) -> eid` | Write a store entry; on the journal, append a `note` (the journal is never overwritten) |
| Data | `QUERY` | `(name, query, k) -> hits[]` | Keyword, vector, or hybrid search over a store or the journal |
| View | `COMPOSE` | `(plan) -> view` | Build an immutable view from a plan; report per-segment tokens and cache state |
| Execute | `INFER` | `(view, model, constraints) -> receipt` | One model call on a view. Result shape in the next section |
| Execute | `CALL` | `(tool, args, mode=async) -> receipt` | Invoke a tool. Async unless `mode=block` |
| Execute | `AWAIT` | `(receipts[], all \| any, timeout?) -> results[]` | Block until receipts complete or the timeout passes |
| Execute | `CANCEL` | `(receipt)` | Abandon an in-flight operation; result status becomes `cancelled`; tokens spent are lost |
| Agents | `SPAWN` | `(spec) -> agent` | Start an environment-driven agent loop. The agent handle is also a receipt: awaiting it yields the agent's return value |
| Agents | `SIGNAL` | `(agent, sig)` | `pause`, `resume`, `kill`, or `warn` (budget). Delivered by the environment |
| Events | `ON` | `(op, phase, handler) -> sub` | Subscribe at Enter, Send, Complete, or Exit of any instruction, or to an environment event |
| Events | `OFF` | `(sub)` | Unsubscribe |
| Meta | `CAPS` | `() -> caps` | Namespace visible to the caller, tools and their specs, models, extensions, `CTXLEN` per model, budget remaining |

The user is not an instruction group. The user is a tool named `user` in the prime agent's namespace: `CALL("user", output)` shows output and returns the user's reply.

**What left the base, and how it is recovered:** contexts and their edits become plan manipulation in the caller (plans are plain data); compaction is a plan that references a summary instead of the original events; forking is copying a plan; `RUN` becomes `SPAWN`; human I/O becomes the `user` tool.

## INFER, decode, and tool results

`INFER` is the atomic instruction: it runs to completion or is cancelled, never interrupted. Parsing the model's raw output is implementation (the decoder); the **outcomes** of decoding are architectural.

**Result shape.**

```
{
  calls:   [{id, tool, args}],              # well-formed calls to tools in the namespace
  rejects: [{kind, raw, detail}],           # candidate calls that failed to decode
  text:    string | null,                   # plain text outside any call
  stop:    "calls" | "text" | "length" | "cancelled",
  cost:    {input, cached_input, output}
}
```

**Reject kinds:** `unparseable` (not valid call syntax or JSON), `unknown_tool` (name not in the namespace), `schema` (arguments fail the tool's schema), `denied` (tool exists but capabilities forbid it).

**Plain text.** Every turn should end in calls. Text with no calls is an **illegal instruction**, with one decode-layer allowance a caller may enable (`constraints.text_as`): treat it as a call to `user` (prime agents) or `return` (subagents). The allowance exists because models are trained to end turns with prose.

**Repair policy** (`constraints.repair`), applied when `rejects` is non-empty or text is illegal:

| Mode | Behavior | Trap kind |
| --- | --- | --- |
| `invisible` | Environment appends the errors and re-infers, up to `retries` times; the caller sees only the final result | Invisible (still journaled and observable) |
| `contained` | Return the result with `rejects` to the caller or an `ON(INFER, Complete)` handler to decide | Contained |
| `fatal` | Fail the receipt with status `error` | Fatal |

Default: `invisible` with one retry, then `contained`.

**Tool result shape.** Every `CALL` result, whatever the tool:

```
{
  status:  "ok" | "error" | "cancelled" | "timeout" | "unknown",
  content: any,                             # rendered into views as the tool's output
  error:   {kind, detail} | null,           # kinds include denied, invalid_args, tool_failure
  effect:  "read" | "idempotent" | "side_effect"   # copied from the tool's declared spec
}
```

**Abort semantics.** A handler at `ON(CALL, Enter)` may deny a call: the result is `status: error, error.kind: denied`, and the agent continues and sees the reason. Stopping the whole agent is a separate act: `SIGNAL(agent, kill)`. This resolves port-test case A.

## Async calls, receipts, and interrupts

**Receipts.** `INFER`, `CALL`, and `SPAWN` return a receipt immediately. A receipt is a serializable record (`{rid, op, issued_eid, status}`), not an in-memory object, so it survives a restart. Its result is the journal event that completes it.

**Blocking is explicit.** A program blocks with `AWAIT`. A model blocks by calling a tool with `mode: block`, or by calling `user`, which always blocks the agent's loop because a user reply is the result it waits for.

**Interrupts.** Any completion or message for an agent that is not currently awaiting it becomes an **interrupt**: an event queued for that agent. The environment delivers interrupts only at instruction boundaries, meaning between one `INFER` and the next. It never splits an inference call. On delivery, the interrupts are added to the agent's next view as `interrupt` events and journaled.

**Coalescing.** Before reinvoking a model, the environment waits up to a coalescing window (default a few seconds, configurable per agent) for further fast completions, and delivers everything that arrived as one batch. This keeps async-by-default from multiplying inference calls. Waiting on a blocking call or the user ignores the window.

**Delivery order (provisional).** Within a batch: user messages first, then errors and timeouts, then results in completion order. A user message arriving while the agent is mid-inference is delivered at the next boundary; the in-flight call is not cancelled unless a handler chooses to `CANCEL` it.

**Late results.** A result for a receipt its agent has stopped caring about is still journaled and delivered. Agents ignore what they don't need, and v0.3 does not drop events silently.

**Not in v0.3:** masking (deferring interrupts during a critical sequence) and priority classes beyond the order above.

## Agents

An **agent** is a model driving its own sequence of instructions, with the environment supplying the loop, the way silicon supplies the fetch–execute cycle. A **program** that calls `INFER` itself is a workflow: it uses the model as a coprocessor. Both are first-class; a program can also emulate the agent loop when it needs custom control.

**SPAWN spec.**

| Field | Meaning |
| --- | --- |
| `plan` | Initial composition plan: shared prefix, mission, initial material |
| `model` | Model to run, with its `INFER` constraints |
| `namespace` | Names the agent can see: tools, stores, channels. This is its capability set |
| `budget` | Tokens, wall-clock time, and money; scoped budgets are shared with children |
| `composer` | The program that builds the agent's view each turn. Default: append-all. This is where context management plugs in |
| `coalesce` | Coalescing window for interrupts |

**The environment loop for one agent**, repeated until the agent returns or is killed:

1. Collect new events for the agent: results, interrupts, user replies.
2. Run the agent's `composer` to produce a plan; `COMPOSE` it into a view.
3. `INFER` on the view; apply the repair policy to any rejects.
4. Dispatch the calls: async by default, blocking where requested.
5. If any call blocks, or the agent called `user`, wait for it. Otherwise wait for the coalescing window, then loop.

**Ending a turn.** A subagent ends by calling the built-in `return(value)` tool; the value completes the agent's receipt. The prime agent never returns. It hands control to the user by calling `user`, and its loop waits on that call, possibly for days, durably. Plain text with no call follows the decode allowance in the INFER section. A handler on `ON(return, Enter)` or `ON(user, Enter)` can refuse the end of a turn and send the agent back to work, which is Claude Code's Stop hook (port-test case B).

**Default subagent.** Unless granted more, a spawned agent sees only its own tools and the `return` tool: an isolated, nondeterministic function call. Asking the parent or user, spawning grandchildren, and peer channels are grants in its namespace (see Sketched areas).

## Composition

A **plan** is plain data owned by the caller: an ordered list of segments. `COMPOSE` resolves it into a **view**, the exact token sequence an inference call will see. Editing a context means editing a plan; the journal underneath never changes.

**Segment types.**

| Segment | Content | Typical position |
| --- | --- | --- |
| `prefix(id)` | A named canonical prefix: base instructions and tool definitions shared across agents | First, for cache sharing |
| `mission(text)` | The agent's instructions and goal | After the prefix |
| `events(selector, render)` | Journal events by range, agent, kind, or query; rendered in full or as references | Middle and tail |
| `store(name, key)` | A store entry, such as notes or a plan file | Anywhere |
| `ref(eid, summary_eid)` | Stand-in for spilled content: a summary plus the id to reload it from | Where the original was |
| `text(content)` | Literal content | Anywhere |
| `interrupts()` | Pending interrupts for the agent | Tail |

**View report.** `COMPOSE` returns, with the view, the token count per segment, the total against `CTXLEN`, and the length of the leading run of segments expected to hit the provider's cache, based on what this environment recently sent. That is the cost visibility an allocator needs to price its choices.

**Cache rule.** A view's cacheable length is its longest prefix identical to a recently sent view for the same model. So composers should put stable segments first, volatile segments last, and change the middle rarely and in batches. Agents sharing one `prefix(id)` share cache.

**Spill and reload.** Replacing `events(...)` with `ref(eid, summary_eid)` spills content; the summary is an `INFER` result already in the journal. Reloading is putting the original selector back. Nothing is lost, because the journal keeps the original. This is how compaction works in v0.3: a choice of view, not a destruction of history.

**The composer.** The default composer reproduces today's behavior (prefix, mission, then every event since spawn). Context management, the register allocation of this system, is any other composer: working-set selection, retrieval over the journal, summaries of old turns. The composer runs as a program, so the model can also be given composer operations as tools to manage its own view.

## Sketched areas

These are directions, not specs. Each already has a hook in the core so the first build doesn't rule it out.

| Area | Direction | Hook already in the core |
| --- | --- | --- |
| IPC channels | Channels are names in the namespace. Send with `PUT`, receive with `READ` or as interrupts. Kinds: RPC to parent (`ask_parent`), asks to the user (direct, or routed up as a vertical trap the parent may answer, batch, or forward), named peer channels, shared state with **leases** for exclusive ownership. Every wait carries a timeout or budget | Namespace, interrupts, `PUT`/`READ` |
| Namespaces | Per-agent namespaces mounted by the parent at `SPAWN`; what an agent can name is what it can touch (Plan 9 style). Generic verbs make journaling, permissions, tracing, and replay uniform | `SPAWN.namespace`, `CAPS` |
| Authority | Every event carries an authority level; content keeps its level when summarized or recomposed; instructions found in low-authority content are data, never policy | `authority` field on events |
| Effect classes | Tools declare `read`, `idempotent`, or `side_effect`, plus optional compensations, so retries and rewinds know what is safe; supervisors (OTP-style) can restart failed subagents | `effect` on tool specs and results; `unknown` status on recovery |
| Cache hints | Plans may carry `keep_warm_until`; the scheduler honors it with keepalive calls or longer provider cache lifetimes when expected reuse justifies the cost, and runs siblings sharing a prefix close together in time | View report, shared `prefix(id)` |
| Extensions | v0.2's KV-level and token-level extensions (`K`, `T`) and constrained decoding carry forward unchanged as optional capabilities, advertised in `CAPS` | `CAPS` |

## Acceptance

The twenty port-test cases from the workbook, judged on paper against v0.3. Expected: 12 pass on the core, 7 are partial because they depend on sketched areas, and 1 reveals a new gap. The first executable test should confirm the passes on the core cases.

| Case | Behavior | Expected | How v0.3 handles it, or what's missing |
| --- | --- | --- | --- |
| A | Block a dangerous command; agent continues | Pass | `ON(CALL, Enter)` denies; result `error.kind: denied` |
| B | Refuse to stop until work is done | Pass | Handler on `ON(user, Enter)` or `ON(return, Enter)` |
| C | Back up before compaction | Pass | Compaction is a view change; the journal already keeps everything; invisible traps are journaled |
| D | Load project instructions at session start | Partial | A `store(...)` segment in the prime's plan works; authority labeling is sketched |
| E | Approval survives a process restart | Pass | Serializable receipts; recovery by journal replay |
| F | Rewind and retry | Partial | Rewinding the view is a plan change; undoing side effects needs compensations (sketched) |
| G | Hand off to a specialist | Pass | Handoff dissolves: spawn or recompose with a new mission over the same journal |
| H | Minimal four-tool loop | Pass | Default agent loop, or a program loop over `INFER` and `CALL` |
| I | Tool asks the model or user (MCP sampling, elicitation) | Partial | Needs channels granted to tools (sketched) |
| J | Malformed tool call | Pass | `rejects[]`, repair policy |
| K | Parallel calls; user corrects mid-generation | Pass | Async calls; the correction is an interrupt at the next boundary |
| L | Prime agent with three async subagents | Pass | Receipts, interrupts, coalescing, provisional delivery order |
| M | Six coders coordinate on one file | Partial | Peer channels and leases (sketched) |
| N | Question routed from subagent to parent to user | Partial | Routed asks (sketched) |
| O | Swap the prime agent's model mid-session | Pass, with caveat | Recompose for the new model; opaque blobs from the old model can't be replayed to it |
| P | Spill and reload a detail from 200 turns ago | Pass | `ref` segments, `QUERY` over the journal, a working-set composer |
| Q | Five siblings share a cached prefix | Partial | Shared `prefix(id)` passes; keepalive is sketched |
| R | Crashed subagent retried without duplicate writes | Partial | Idempotent re-issue and `unknown` status are specified; compensations and supervision are sketched |
| S | Prime agent waits three days on the user, survives restart | Pass | `user` call is a durable receipt |
| T | Agent hardens a routine into a program it reuses | **Gap** | v0.3 has no way to install a model-written program as a tool in a namespace |

## Open questions

- [ ] **Programs as tools (case T).** How does a model-written program get installed into a namespace as a callable tool, sandboxed, versioned, and revertible? This is the core mechanism for the self-adapting agent and the biggest gap in v0.3.
- [ ] **Interrupt masking and priority.** Is the provisional delivery order enough, or do agents need to defer interrupts during critical sequences?
- [ ] **Coalescing window.** Fixed, adaptive, or chosen per call? Needs measurement from the first build.
- [ ] **Composer cost.** A composer that calls `INFER` (for summaries or retrieval) spends tokens to save tokens. How is its cost charged and bounded?
- [ ] **Concurrency on stores.** What do concurrent `PUT`s from sibling agents see? Leases cover exclusive ownership; ordinary writes need a stated consistency rule.
- [ ] **Opaque provider state.** How much portability is lost to model-bound blobs in practice?
- [ ] **Authority levels.** A fixed ladder (provider, organization, project, user, tool, retrieved) or a lattice?
- [ ] **Does `SIGNAL` survive?** If kill is `CANCEL` on the agent's receipt, and warnings are events, `SIGNAL` may reduce to pause and resume.

## Related documents

- **Harness ISA — Draft v0.1** and **Draft v0.2**: earlier drafts, kept as history.
- **Harness ISA — Workbook**: decisions, architecture sketch, prior-art smorgasbord, port test (cases A–T), requirements R1–R16, and raw notes from the loops session.
- **Reference: CPU ISA Design** and **Reference: Harness Principles**: the inspiration and the target.
- **Handoff: Harness ISA to Claude Code**: instructions for continuing this work in Claude Code.
