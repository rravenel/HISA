# Reference: JIT Compilation Principles

Oct 20, 2026 · @Rene Ravenel

Third reference for the Harness ISA project, after CPU ISA design and harness principles: how just-in-time compilers decide what to compile, when, and how they stay correct, applied to agents that move work from the model into code.

## Why JIT, and what "self" means

**"Self" is looser than it sounds.** A script that hands its own source to a model and overwrites itself is literally self-modifying. That doesn't scale down: an edge device running tiny models lacks the competence to modify itself, let alone improve. What the project actually needs is one harness acting on another:

- An **authoring harness**, with generous hardware and frontier models, builds a **target harness** for a specific use case and environment.
- The target can start as a **fork** of the authoring harness, which then adds, removes, and reconfigures; or **from nothing but the ISA implementation**, built up from scratch.
- At that point the authoring harness is a coding agent whose application happens to be an agent, or more precisely an application built on the ISA that uses language models and follows harness design principles.
- True self-modification survives as one configuration: the target *is* the authoring harness, and the task is improving its own ability to build agents.

**Why JIT is the right lens.** A JIT compiler watches an interpreter run, finds the code that runs most, compiles it to fast native code, and keeps it correct with guards that fall back to the interpreter when assumptions fail. The project's goal has the same shape: a model (slow, expensive, unpredictable) executes a task; the system finds the paths it runs most; those paths become deterministic code; and the model stays available as the fallback. The principle "inference is the tool of last resort" fits a JIT naturally.

**But removal is only half of it.** These are AI applications: the whole point is tasks that need a model. Some inference calls are irreducible (understanding a spoken request, judging an ambiguous case), and those are what make the application worth building. A compiler doesn't only eliminate instructions; it also makes the ones that must run cheaper. So the authoring agent has two goals: **move what can be code into code**, and **make the inference that remains cheaper**: called less often, with fewer tokens per call, and with better odds of being right. Authoring code is justified by either goal, not only by deleting a call. See "Optimizing the inference that stays" below.

Two deployment shapes follow, and compilers already have names for both:

- **JIT:** one harness promotes model behavior into its own code while it runs (in-situ, self-modification in the literal sense).
- **AOT with profile-guided optimization:** the authoring harness profiles a workload with a frontier model as the interpreter, compiles a target harness, and ships it. The edge device is the AOT case, since its tiny model can't serve as a capable interpreter or compiler.

And the self-improving configuration has a name too: a compiler that compiles itself is **self-hosting**, and improving it is **bootstrapping**.

## The mapping

The model is the interpreter, deterministic harness code is compiled code, the journal is the profiler, and the authoring agent is the compiler. Work moves rightward as evidence accumulates, and falls back to the model whenever compiled code meets a case it wasn't built for.

&#91;embedded content: the tier ladder · promotion rightward, fallback leftward, the journal underneath\]

## Core JIT principles and their harness counterparts

Most of these ideas trace to one lineage: the **Self** language project (Ungar, Chambers, Hölzle, 1987–1995), whose adaptive optimization, type feedback, polymorphic inline caches, and dynamic deoptimization became Sun's HotSpot JVM and shaped every modern JavaScript engine. The project's name is a coincidence worth enjoying.

| Principle | In a JIT | In a self-adapting harness |
| --- | --- | --- |
| **Interpret first** | Every method starts in the interpreter: cheap to start, slow to run | Every new task starts with the model driving: flexible, expensive |
| **Adaptive optimization** | Most execution time is spent in a small part of the code, so find the hot spots while running and optimize only those | Most tokens are spent on a few recurring patterns; find them in the journal and harden only those |
| **Tiered compilation** | HotSpot runs five tiers: interpreter, C1 without profiling, C1 with basic profiling, C1 with full profiling, C2 fully optimized. Code moves up tiers as it proves hot; thresholds adapt to compiler load | Model-driven → scratch script → installed tool → harness program; thresholds based on recurrence, token cost, and authoring capacity |
| **Profiling and type feedback** | Counters and profiles record what actually happens (which types arrive at a call site) and guide what to compile and how | The journal records which tools the model picks, with what argument shapes, in what order, and how often each succeeds |
| **Inline caches** | Remember the result of a method lookup at the call site; polymorphic inline caches keep several. Their side effect: free per-site type profiles | **Decision caches:** remember what the model decided in a given situation. A situation where the model always decides the same way is "monomorphic" and a prime candidate for code |
| **Speculation and guards** | Compile for what has been observed, guarded by checks that the assumptions still hold | Promoted code declares when it applies (input shape, intent, data ranges) and checks before acting |
| **Deoptimization** | When a guard fails, rebuild the interpreter's state and continue there, transparently | When a guard fails, compose a view from the journal and hand the step back to the model |
| **On-stack replacement (OSR)** | Switch a running loop between interpreted and compiled code mid-execution | Switch a running task between model-driven and code-driven mid-flow; the journal makes any point's state recoverable |
| **Method vs. tracing JITs** | Method JITs compile hot functions; tracing JITs record a hot path through a loop, compile that linear trace, and handle divergence with side exits, which become side traces when taken often | Method: harden a whole skill or tool. Trace: harden a recurring *sequence of turns*, with side exits back to the model; frequent exits become their own compiled branches |
| **Partial evaluation (Futamura projections)** | Specializing an interpreter to a fixed program yields a compiled program (first projection); specializing the specializer yields a compiler (second); and a compiler-compiler (third). PyPy and GraalVM's Truffle use the first projection in practice | Specializing a general agent to a fixed task yields a task-specific application: the authoring agent is a specializer. Improving the authoring agent at its own job resembles the higher projections |
| **Meta-tracing** | PyPy traces the *interpreter* running a program, not the program, so one JIT serves any language written in its framework | Profile the harness running the model, not the model, so one authoring agent serves any application built on the ISA |
| **AOT with profile-guided optimization** | Compile ahead of time using profiles from representative runs | The edge case: profile on the authoring side, compile, ship |
| **Code cache management** | Compiled code takes memory; cold or invalidated code is evicted | Installed programs take namespace and attention; low-hit or high-fallback programs are retired |
| **Warm-up** | Programs run slowly until hot code is compiled | Early runs spend tokens on interpretation and profiling before any savings; there is a break-even point |
| **Invalidation** | Compiled code depends on assumptions (e.g. a class hierarchy) and is discarded when they change | Programs depend on tool schemas, APIs, and policies; a change invalidates them back to the model |
| **Bootstrapping and self-hosting** | A compiler written in its own language compiles successive versions of itself | The authoring agent improving the code it is made of |

## Applying them: mechanisms for the authoring agent

1. **Profile from the journal.** No new instrumentation: the journal already records every call, argument, result, cost, and outcome. The authoring agent mines it for three kinds of hotness: recurring tool-call *sequences* (trace candidates), recurring *decisions* in similar situations (decision-cache candidates), and recurring *transformations* the model performs on data (function candidates). Each candidate carries its frequency and its tokens per occurrence, so its expected savings are known before anything is written.
2. **Decide with a break-even rule.** Promote when (occurrences × tokens saved per occurrence) exceeds (authoring cost + verification cost + expected fallback cost). Thresholds, as in HotSpot, should adapt to how busy the authoring agent is.
3. **Climb tiers, never jump.** Model-driven → scratch script → installed tool → harness program. Each tier gathers the evidence for the next: a tool the model calls successfully a hundred times is better evidence than a pattern seen in the journal.
4. **Every compiled path has a guard and a fallback.** The guard states when the code applies; on a miss, the environment composes a view from the journal and hands the step back to the model (deoptimization). Because the journal reconstructs any point's state, mid-flow switching (OSR) is cheap.
5. **Count fallbacks; demote on drift.** A guard that misses too often means the speculation was wrong or the world changed. Demote the program a tier, the way JITs discard and recompile code that deoptimizes repeatedly.
6. **Prefer traces for agents.** Agent work is sequential and branchy, much like the loops tracing JITs were built for. Compile the common path through a task (look up the order, check the policy, draft the reply) and let rare branches exit to the model; branches that exit often become compiled side paths.
7. **Specialize, then verify.** The authoring agent's core move is partial evaluation: take the general agent and the fixed facts of a use case (its tools, policies, data shapes, user population) and produce a specialized application. Verification before registration replaces the semantic-preservation proofs real partial evaluators rely on.
8. **Edge targets are AOT.** Profile on the authoring side using the frontier model as the interpreter on representative workloads; compile the target harness; ship it with the tiny local model as the fallback interpreter. Since that fallback is weak, guards on edge targets should be conservative, and fallbacks should escalate to a human or a hosted model when one is reachable.
9. **Invalidate on dependency change.** Programs declare what they assume (tool schemas, API versions, policy versions). When an assumption changes, dependents drop back to the model until re-verified.
10. **Retire cold code.** Programs that are rarely hit or frequently fall back are evicted from namespaces, like a code cache, keeping the model's tool list and the authoring agent's maintenance load small.

## Optimizing the inference that stays

For a call that can't be removed, three levers remain: **frequency** (how often it runs, set by the architecture or algorithm around it), **tokens per call** (set by the prompt, the data, and the output format), and **quality per call** (set by what the model is given to work with, which also reduces retries). Classic compiler optimizations map onto all three.

| Compiler optimization | What it does to instructions | Inference counterpart | Lever |
| --- | --- | --- | --- |
| Loop-invariant code motion | Moves a computation that doesn't change out of a loop | Ask once per task instead of once per item, when the answer doesn't depend on the item | Frequency |
| Common subexpression elimination, memoization | Computes a repeated expression once | Deduplicate identical calls; reuse a cached decision for an identical situation | Frequency |
| Vectorization (SIMD) | One instruction processes many data elements | Batch: classify or extract fifty items in one call instead of fifty calls | Frequency, and tokens via a shared prefix |
| Inlining, call elimination | Removes call overhead for small functions | Fold a model step into a neighboring one, e.g. extract and decide in one call | Frequency |
| Constant folding, partial evaluation | Precomputes what is known at compile time | Specialize the prompt: bake in facts known at authoring time, delete instructions that don't apply to this use case | Tokens |
| Register allocation | Keeps only live values in scarce registers | Composition: give the model concise, relevant data, not everything | Tokens, quality |
| Strength reduction | Replaces an expensive operation with a cheaper equivalent | Route to a smaller or cheaper model role when it is good enough | Cost per token |
| Narrower data types | Uses the smallest type that holds the value | Constrain output: schemas, enums, grammars; shorter outputs, fewer repairs | Tokens, quality |
| Better instruction set | Richer instructions do more per step | Better tools for the model to call, so it needs fewer turns and less reasoning | Frequency, quality |
| Hardware acceleration, custom instructions | Dedicated silicon for a hot operation | A small model fine-tuned or distilled for one recurring call: the inference version of a custom instruction | Cost per call |

The last row is the edge story. A voice-controlled thermostat on a local stack such as Cactus's Whistle (speech-to-text) followed by Needle (speech-to-function-call) can never drop the models: understanding speech *is* the task. What the authoring agent can do is everything else in the table: a tool set reduced to the thermostat's few functions in a stable, cached prefix; output constrained to those function schemas; deterministic range and safety checks on the resulting call (no setting the house to 95 degrees without confirmation); a confirmation turn only when the guard is unsure; and, where available, a model specialized for exactly this call. The model stays on the path, smaller, shorter, and better supported.

**Measuring it.** Every lever shows up in the journal: calls per task, tokens per call by type, retries and repairs per call, and success rate. The authoring agent's job on irreducible calls is to move those numbers, using the same profile, specialize, verify, and promote loop as for code.

## Risks the JIT world already learned about

| Risk | In JITs | In a self-adapting harness | Mitigation |
| --- | --- | --- | --- |
| Over-speculation | Code compiled for types seen during warm-up is wrong for later workloads | Programs fit to the profiled workload fail on new users, seasons, or products: overfitting | Guards, fallback counting, demotion; profile on representative and adversarial workloads |
| Deoptimization storms | Repeated deopt and recompile cycles cost more than interpreting | Thrashing between code and model on a borderline pattern costs more tokens than leaving it with the model | Back-off: a program demoted twice stays demoted until new evidence |
| Silent wrongness | A miscompile produces fast, wrong answers | Promoted code handles a case it shouldn't and nobody notices, because the model is no longer looking | Guards plus sampled audits: periodically route a fraction of guarded cases to the model and compare |
| Code bloat | Large code caches; tiering multiplies compiled versions | Tool sprawl crowds the model's context and the maintenance budget | Eviction of cold programs; namespace limits |
| Warm-up cost | Slow start before compilation pays off | Tokens spent profiling and authoring before savings arrive | Break-even rule; AOT for known workloads |
| **Trusting trust** | Ken Thompson's 1984 Turing lecture showed a self-hosting compiler can carry a hidden behavior into every future version, invisible in its source | A self-improving authoring agent could carry a flaw or a bad incentive into every target it builds and every version of itself | Keep an independent verification lineage: conformance tests and guardrail policy the authoring agent cannot modify, human review of changes to the authoring agent itself, and cross-checking outputs against an independently built harness (the idea behind David Wheeler's diverse double-compiling) |

The last row is the one to take most seriously. It's the compiler-world version of the rule already in the workbook: an agent being optimized must never be able to rewrite its own guardrails or authority rules.

## Impact on the ISA vs. the harness built on it

Most of the JIT machinery belongs in the **authoring agent**, the harness being built on top of the ISA. The ISA's job is to make that machinery possible and cheap, the way a CPU provides performance counters and memory protection but no compiler.

**Already provided by the v0.3 design and v0.4 candidates:**

- The journal as the profiler and as the state source for fallback and mid-flow switching.
- One interface for all callers (R18, R19), so promoting code is re-registration, not a rewrite.
- Sandboxed sessions and installed programs (R17, R20), the code cache.
- Promotion stages, guarded replacement with fallback, and privilege levels (R37–R39).

**New ISA-level needs this exploration adds** (v0.4 candidates):

- **Declared assumptions and invalidation (R40).** A registered program lists what it depends on (tool schema versions, API versions, policy versions). The environment invalidates it, routing back to the model, when one changes.
- **Fallback as a first-class outcome (strengthens R38).** A guard miss is a standard result shape the environment knows how to handle (compose a view, hand to the model, journal it), not something each program reimplements.
- **Sampled audit routing (R41).** The environment can route a configurable fraction of a program's guarded cases to the model as well, for comparison, so silent wrongness surfaces.

**Belongs in the authoring agent, not the ISA:** hotness mining, break-even decisions, tier policy, trace selection, specialization strategy, demotion policy, eviction policy, and AOT workflows for edge targets.

## Sources

- [Hölzle, Adaptive Optimization for Self (PhD thesis, 1994)](https://bibliography.selflanguage.org/urs-thesis.html): type feedback, adaptive optimization, dynamic deoptimization, polymorphic inline caches
- [Hölzle, Chambers, Ungar, Polymorphic Inline Caches (ECOOP 1991)](https://bibliography.selflanguage.org/pics.html)
- [Ungar & Smith, Self (HOPL 2007)](<https://worrydream.com/refs/Ungar_2007_-_Self_(HOPL).pdf>) and [The influence of Self](https://dubroy.com/blog/self/)
- [How tiered compilation works in OpenJDK (Microsoft)](https://devblogs.microsoft.com/java/how-tiered-compilation-works-in-openjdk/) and [HotSpot tiered threshold policy source](https://android.googlesource.com/platform/libcore/+/6a85e1454d4/src/hotspot/share/compiler/tieredThresholdPolicy.hpp)
- [Tracing JITs for Dynamic Languages (Northeastern PRL)](https://prl.khoury.northeastern.edu/blog/2017/03/15/tracing-jits-for-dynamic-languages) and [Mike Pall on LuaJIT 2.0 trace compilation](https://gist.github.com/jmpnz/fb8a1f2c9c0e70b4d2b0cc6cb5ddec25)
- [Partial evaluation and the Futamura projections](https://en.wikipedia.org/wiki/Partial_evaluation) and [Fallin & Bernstein, Partial Evaluation, Whole-Program Compilation (PLDI 2025)](https://arxiv.org/html/2411.10559v1)
- From general knowledge, not fetched this session: Thompson, "Reflections on Trusting Trust" (1984 Turing Award lecture); Wheeler, diverse double-compiling (2009 dissertation).
