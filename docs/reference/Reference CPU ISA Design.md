# Reference: CPU ISA Design

Oct 19, 2026 · @Rene Ravenel

Source of inspiration for the Harness ISA project: how CPU instruction sets are defined and why.

## What an ISA is

An ISA is a contract with two faces, and each face binds a different implementer.

- **Upward face (software).** What compilers, assemblers, operating systems, and programmers may rely on: which instructions exist, what each does to architectural state, and what happens on error. Software written to this face runs on any conforming chip.
- **Downward face (hardware).** What every implementation must deliver, and nothing more. RISC-V's spec says it should be read as the software-visible interface to many implementations, not the design of one chip.

The art is in what the contract leaves out. Anything it pins down constrains every future implementation; anything it leaves open (pipeline depth, cache size, decode strategy) is free for implementers to innovate on. RISC-V marks deliberately unconstrained behavior with an explicit term, UNSPECIFIED.

*For the Harness ISA:* both faces are software. The upward face binds skills, programs, and the model's tool-calling behavior; the downward face binds harness implementations and inference engines. Output parsing, retry loops, and context assembly all sit below the line, which is why the contract must specify their outcomes rather than their code.

## The layer stack

The ISA is not one interface but the shared core of several. RISC-V names each layer's contract: an application talks to its execution environment through an **ABI** (user ISA plus environment calls), an operating system talks to its supervisor environment through an **SBI**, and a hypervisor talks to its environment through an **HBI**. Using one SBI across environments lets a single OS binary run on any of them.

&#91;embedded content: layer stack · RISC-V software stack beside its harness analog\]

Read row by row, the harness column shows where drafts v0.1 and v0.2 sit: at the ABI line. The line below it, between a harness and the outer system that governs it (an enterprise policy layer, a sandbox host, a parent harness running a child), has no standard yet. That is where permissions and audit policy would live.

## What an ISA specifies

A complete ISA specification covers seven areas. The harness column is a first guess at each analog, to be tested against the Harness Principles reference.

| Area | What the spec defines | RISC-V example | Harness analog |
| --- | --- | --- | --- |
| Architectural state | Everything software can observe: registers, program counter, control/status registers | 32 integer registers, CSRs | Context, mission, caps, budget, handles |
| Instructions | Each operation's effect on state, operands, and failure cases | \~40 base integer ops plus system ops | HB instructions |
| Memory model | Address space, alignment, and the order in which other harts observe writes | RVWMO weak ordering, optional total store order (Ztso) | Stores and concurrent forks; v0.2 has nothing here |
| Traps | Exceptions, interrupts, and their four effects (contained, requested, invisible, fatal) | ECALL, illegal instruction, page fault | Tool call, rejected call, retry, budget kill |
| Privilege | Modes and which instructions each may execute; trap routing between modes | U, S, M modes; SBI between OS and firmware | Model vs. skill program vs. harness vs. user policy |
| Encoding | Bit layout of each instruction; reserved and custom space | Fixed 32-bit base, 16-bit compressed, custom opcode slots | Tool-call wire format; deliberately left to the HEI |
| Discovery | How software learns what is present | `misa` register, profile names | `CAPS` |

Two areas matter most for the harness project and are thinnest in v0.2: the **memory model** (what happens when forks write the same store) and **privilege** (who may call what, and where a trap is routed).

## Design principles and their reasoning

The classic four come from Patterson and Hennessy's textbooks; the rest come from RISC-V's commentary and from 50 years of mistakes.

| Principle | Reasoning | Harness reading |
| --- | --- | --- |
| Simplicity favors regularity | Uniform formats (MIPS always has three operands in the same fields) make decode simple and fast | One handle model and one result shape for every slow instruction |
| Smaller is faster | More registers or larger structures lengthen the clock cycle | A smaller base is cheaper to implement, verify, and fit on a reference card |
| Make the common case fast | Optimizing the common case beats optimizing the rare one; knowing which is which needs measurement | Measure real harness traces before adding instructions; fuse what is common |
| Good design demands good compromises | MIPS varies formats while keeping one instruction length | Portable baseline vs. engine power, resolved by profiles |
| Amdahl's Law | Speeding one part has diminishing returns on the whole | Inference dominates; harness-side speedups matter mostly at scale |
| Don't over-architect for one microarchitecture | Delay slots and trap barriers aged badly | Keep vendor formats, reasoning syntax, and tokenizers out |
| Separate unprivileged from privileged | RISC-V's privileged design could be replaced without changing the unprivileged ISA, or even the ABI | Keep what programs call separate from how the harness governs them |
| Plan for address growth | Address size is the hardest mistake to undo | Context length is a parameter |
| Mark the unspecified | RISC-V names deliberately unconstrained behavior | Say explicitly what HB leaves open |

Two privilege ideas from the RISC-V privileged spec deserve separate mention. Traps that raise privilege are *vertical* and those that stay at the same level are *horizontal*, and trap routing between layers is configurable. And the spec separates the privilege level code is *written for* from the mode it *runs in*: a supervisor-level OS can run in user mode under a hypervisor that traps and emulates its privileged actions. A harness analog would let a skill written to command the harness run unmodified under a stricter outer harness that traps and vets every call.

## Evolution and compatibility

ISAs live for decades, so how they grow matters as much as where they start. Three strategies are on record.

- **x86: never break old binaries.** Every generation adds; nothing leaves. The manual reached about 3,600 pages, and modern chips translate the complex ISA into simpler internal operations. That translation is worth paying for only because of a huge legacy code base.
- **RISC-V: freeze and layer.** The base and each extension are kept constant, and new instructions arrive as new optional extensions instead of a new ISA version.
- **RISC-V profiles: bound the optionality.** Extensions alone risk fragmentation, so RISC-V International bundles them into versioned profiles. RVA23, ratified in 2024, made vectors and the hypervisor extension mandatory. Its stated goal is a large set of guaranteed extensions and only a few discoverable, coarse-grained options, and it explicitly rejects a minimal base plus many fine-grained extensions. Software ecosystems then pick a profile generation as their baseline.

*For the Harness ISA:* v0.2's alphabet of small extensions is closer to what RVA23 rejects. A better target is few, coarse profiles that grow by generation, such as "HG-27" mandating what most harnesses already ship.

## Reference documentation

| Document | Use it for |
| --- | --- |
| [RISC-V Unprivileged ISA, Introduction](https://docs.riscv.org/reference/isa/v20240411/unpriv/intro.html) | Goals, EEI, harts, trap kinds, UNSPECIFIED, base vs. extensions |
| [RISC-V Privileged ISA, Introduction](https://docs.riscv.org/reference/isa/v20240411/priv/priv-intro.html) | ABI/SBI/HBI layering, privilege modes, vertical vs. horizontal traps |
| [RISC-V Extension Naming](https://docs.riscv.org/reference/isa/unpriv/naming.html) | Standard, Z, and X naming; the G bundle |
| [RISC-V Extending the ISA](https://docs.riscv.org/reference/isa/unpriv/extending.html) | Standard, reserved, and custom encoding space |
| [RVA23 Profiles](https://docs.riscv.org/reference/rva23/rva23-profiles.html) and [rationale](https://docs.riscv.org/reference/rva-profile-overview-body.html) | Why profiles bound optionality |
| [Asanović & Patterson, Instruction Sets Should Be Free](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2014/EECS-2014-146.pdf) | Base-plus-extension case, classic ISA mistakes, base instruction table |
| [Patterson, 8 Great Ideas in Computer Architecture](https://www.elsevier.com/zh-cn/connect/8-great-ideas-in-computer-architecture) | Common case, abstraction, dependability |
| [Hardware design principles summary](https://dra.cs.southern.edu/HardwareDesignPrinciples) | The four MIPS-era principles with examples |
| [Canonical on RVA23](https://canonical.com//blog/risc-v-profiles-why-is-rva23-significant) | How an OS vendor adopts a profile as its baseline |
