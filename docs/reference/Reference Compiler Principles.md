# Reference: Compiler Principles

Oct 20, 2026 · @Rene Ravenel

How compilers turn intent into machine code through layers of representation, and where the same structure appears in a harness.

## The compiler pipeline and why it is split

A modern compiler has three parts:

- **Front end:** parses a source language and checks it, producing an intermediate representation (IR). One per source language.
- **Middle end:** analyzes and transforms the IR with a sequence of optimization **passes**, each preserving meaning. Shared by all languages and targets.
- **Back end:** lowers the IR to a specific machine: instruction selection, register allocation, instruction scheduling, then encoding. One per target.

The split exists for economics. With *M* source languages and *N* targets, direct translation needs *M × N* compilers; an IR in the middle needs *M + N* components, and every optimization written once in the middle end benefits all of them. LLVM is the canonical example: Clang, Rust, Swift, and many others share one middle end and many back ends.

*Harness reading:* the same economics apply to agent programs and models. Many kinds of agent program (skills, workflows, model-written code) and many targets (hosted models with different tool-call formats, local models, engines with KV control) want one representation in between, so that optimizations such as cache-aware layout and context minimization are written once.

## Intermediate representations

A good IR has a few properties, each with a reason:

- **Independent of source and target**, so front and back ends can vary freely.
- **Explicit**: everything the optimizer needs is stated, not implied. Static single assignment (SSA) form, where each value is defined exactly once, makes data flow visible, which is why most modern IRs use it.
- **Structured for analysis**: a control-flow graph of basic blocks, so passes can reason about what runs before what.
- **Typed and verifiable**: a verifier checks the IR is well-formed between passes, so a buggy pass is caught where it happens.
- **Serializable and readable**: you can dump it, diff it, and test passes on it.

**One level isn't enough.** A single low-level IR throws away structure too early: once loops become branches, loop optimizations are hard to recover. **MLIR** (Lattner et al., 2020) answers this with *multi-level* IR: operations from several abstraction levels, grouped into **dialects**, coexist in one representation and are **progressively lowered**, high-level structure first optimized where it is still visible, then translated step by step toward machine code. Later work adds *progressive raising*, recovering high-level structure from low-level code to unlock optimizations.

*Harness reading:* an agent program has natural levels too (intent, plan, view, provider request), and each enables different optimizations. That points to a multi-level IR, the subject of the next section.

## Does the harness have an IR?

Yes: three, and v0.3 already contains the first two without calling them that.

&#91;embedded content: three IRs · the plan lowering path, the journal, and traces\]

**1. The plan is the main IR.** A plan sits between intent (a skill, a program, a model's decision) and the exact token sequence a model sees. It is target-independent (it doesn't know the provider's message format), explicit (segments say what they are), and serializable. Lowering runs in stages, MLIR style:

- *Plan level:* declarative segments. Passes here: deduplicate, replace events with summaries, reorder for cache reuse, drop resolved errors, enforce authority labels. This is where the database-style planner from the survey would live.
- *View level:* concrete content with token counts and cache estimates. `COMPOSE` lowers plan to view.
- *Provider-request level:* the per-model wire format (message roles, tool-call syntax, thinking blocks, cache markers). The **codec is the back end**, one per model family, exactly as compilers have one back end per CPU.

**2. The journal is the canonical, model-neutral IR of history.** Like LLVM IR serving many front and back ends, the journal records what happened independent of any provider's format, which is precisely what makes model swapping possible. Codecs lower *from* it; nothing upstream depends on a provider.

**3. A trace IR for the JIT.** To turn recurring model behavior into code, the authoring agent needs an intermediate form between raw journal events and a program: a normalized trace of ISA calls with the conditions under which each branch was taken. That is where guards come from, and it is the analog of a tracing JIT's recorded trace before compilation. This one is new.

The practical consequence: name the plan format, version it, and give it a verifier, the way compilers verify IR between passes. Then composition passes can be written once and shared by every agent and every model.

## The back-end trio and the middle-end passes

The three back-end concerns raised early in this project are **instruction selection**, **register allocation**, and **instruction scheduling**. With the middle-end passes, they map as follows.

| Compiler stage | What it decides | Harness counterpart |
| --- | --- | --- |
| Instruction selection | Which concrete instructions implement each IR operation; fusing several into one | Which mechanism runs each step (code, tool, small model, large model, subagent); fusing steps into one inference call; the codec choosing a provider's native features |
| Register allocation | Which values live in scarce registers, which spill to memory | Composition: what is in the context window, what spills to the journal or a store, what is cache-pinned |
| Instruction scheduling | Order and overlap of instructions to keep the pipeline busy | Ordering and overlapping calls: parallel tools, prefetch, coalescing, running siblings close together for cache warmth |
| Dead-code elimination | Remove computations whose results are never used | Drop context segments nothing will use; skip tool calls whose results are unused |
| Common-subexpression elimination | Compute a repeated expression once | Deduplicate identical calls and repeated context |
| Loop-invariant code motion | Hoist unchanging work out of loops | Ask the model once per task, not once per item |
| Inlining | Replace a call with the callee's body | Fold a small subagent into its parent's turn, or a tool's logic into a program |
| Constant folding, partial evaluation | Precompute what is known | Specialize prompts with facts known at authoring time |
| Liveness analysis | Determine when each value is last used | Predict which context entries will matter again; the hard, uncertain part of harness register allocation |
| Alias analysis | Determine whether two references can touch the same memory | Determine whether two agents' actions can touch the same resource; decides what may safely run in parallel |

The last row is new to this project and matters for parallel subagents: a scheduler can only overlap work it can prove independent, and tool effect classes plus declared resources are what make that analysis possible.

## Other compiler lessons

- **Separate compilation and linking.** Modules compile independently and a linker binds their symbols. *Harness:* skills and programs are modules; binding a program's tool names to the namespace is linking, and late binding (load on first use) is progressive disclosure.
- **ABIs and calling conventions.** Separately compiled code interoperates only if everyone agrees how arguments, results, and errors are passed. *Harness:* the standard tool result shape, event kinds, and receipt format are the ABI. Getting them stable matters more than any single instruction.
- **Debug information.** Compilers emit a map from machine code back to source lines so optimized code stays debuggable. *Harness:* promoted code should carry a map back to the journal traces and model decisions it was derived from, so a failure in compiled code can be traced to the behavior it replaced. This is provenance for the JIT.
- **Optimization levels.** `-O0` compiles fast and debuggably; `-O3` spends compile time for speed. *Harness:* authoring effort is a dial too: a quick prototype pass versus heavy specialization, chosen per application.
- **Semantics preservation and translation validation.** Every pass must preserve meaning. Verified compilers (CompCert) prove it; translation validators (such as Alive2 for LLVM) check each transformation. *Harness:* the verification gate before promotion, and golden-master tests from approved journal traces, are translation validation for model-to-code replacement.
- **Bootstrapping.** Compilers written in their own language compile their successors; covered in the JIT reference along with its trusting-trust caveat.

## Implications and v0.4 candidates

- **R42 Plan IR.** Name, version, and specify the plan format, with a verifier that checks plans between composition passes (well-formed segments, authority labels present, budget respected).
- **R43 Composition passes as a standard library.** Deduplication, summarization substitution, cache ordering, and dead-segment elimination written once against the plan IR and shared by all composers, as middle-end passes are shared across languages.
- **R44 Codecs as back ends.** Restate R23 in compiler terms: one codec per model family lowers views to provider requests and raises responses back to the architectural outcome. Nothing above the codec depends on a provider.
- **R45 Trace IR.** A normalized form of recurring journal behavior (ISA calls with their branch conditions) that the authoring agent compiles to programs, with source maps back to the originating events.
- **R46 Declared resources for independence analysis.** Tools declare which resources they touch, so the scheduler can prove two agents' calls independent before running them in parallel.

And the framing to keep: the ISA defines the **instructions and the ABI**; the plan IR and journal are the **representations**; composers, codecs, and the authoring agent are the **compiler**. Mixing these layers is how compilers become unmaintainable, and harnesses too.

## Sources

- [Lattner et al., MLIR: A Compiler Infrastructure for the End of Moore's Law (2020)](https://apxml.com/courses/compiler-runtime-optimization-ml/chapter-2-advanced-ml-intermediate-representations/mlir-lowering-paths) and [TensorFlow's MLIR announcement](https://blog.tensorflow.org/2019/04/mlir-new-intermediate-representation.html?hl=lt)
- [Progressive Raising in Multi-level IR](https://hal-univ-tlse3.archives-ouvertes.fr/INRIA/hal-03139764)
- From general knowledge, not fetched this session: Aho, Lam, Sethi & Ullman, *Compilers: Principles, Techniques, and Tools*; Lattner & Adve, LLVM (2004); Cytron et al., SSA form (1991); Leroy, CompCert; Lopes et al., Alive2 (2021, also referenced in earlier research for this project).
