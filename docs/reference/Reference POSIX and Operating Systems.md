# Reference: POSIX and Operating Systems

Oct 20, 2026 · @Rene Ravenel

The rung above the CPU ISA: how operating systems turned hardware into processes, files, and pipes, and what a harness interface should borrow from them.

## What POSIX is and where it sits

POSIX (IEEE Std 1003.1, first published in 1988) standardizes the interface an operating system offers to programs: processes, files, directories, pipes, signals, threads, users and permissions, plus a standard shell and utilities. Like an ISA, it is a contract, not an implementation. Linux, macOS, and the BSDs implement it to varying degrees, and a program written to it moves between them.

In the layer stack, POSIX sits one rung above the ISA. The ISA turns transistors into instructions; the OS turns instructions into **processes that cannot see or damage each other**, and **resources with names**. Programs never touch hardware: they ask the kernel through system calls, which are the one trap instruction the ISA provides for that purpose.

The harness parallel is direct. The Harness ISA's instructions (`INFER`, `CALL`, `SPAWN`, `READ`, `PUT`) are closer to system calls than to CPU instructions: they're requests to an environment that owns the real resources (models, tools, stores) and enforces isolation between agents. So POSIX is arguably the more exact model for what this project is building, and the CPU ISA the model for how to keep it small and evolvable.

## The Unix philosophy and design principles

Doug McIlroy's summary is the best known: write programs that do one thing and do it well, that work together, and that handle text streams, because text is a universal interface. Around it sit several principles that shaped POSIX:

- **Small, composable tools.** Power comes from combining simple programs with pipes, not from large ones. *Harness reading:* small, focused agents and tools composed by programs (12-Factor's factor 10 says the same).
- **A universal interface.** Text streams let any program feed any other. *Harness reading:* natural language is the text stream of agents, which is both their power and the root of prompt injection; structured data at the boundaries is the discipline that tames it.
- **Everything is a file.** One namespace and a few verbs (open, read, write, close) cover disks, devices, pipes, and sockets, so generic tools work everywhere. Already adopted as per-agent namespaces with generic verbs.
- **Mechanism, not policy.** The kernel provides mechanisms (permissions, scheduling, signals) and leaves policy to administrators and programs. *Harness reading:* the ISA provides journaling, guards, budgets, and hooks; what to guard, how much to spend, and when to ask the user are policy, outside the base. This is the OS form of the end-to-end argument.
- **Fail loudly, recover simply.** Small integer exit codes, `errno`, and standard error give a uniform error channel. Already reflected in the standard tool result shape.
- **Worse is better.** Richard Gabriel's essay argued Unix spread because a simple, slightly incomplete design that ships beats a complete, correct one that doesn't. A useful counterweight to over-specifying v0.4.

## Core abstractions and their harness counterparts

| POSIX abstraction | What it does | Harness ISA counterpart | Status |
| --- | --- | --- | --- |
| Process, PID, parent PID | Isolated unit of execution with identity and lineage | Agent with id and parent, started by `SPAWN` | v0.3 core |
| `fork` + `exec` + `wait` + `exit` | Create a child, give it a program, wait for its exit status | `SPAWN` (spec includes plan and namespace), `AWAIT` on the agent's receipt, `return` | v0.3 core |
| File descriptor | A small integer naming an open resource; inherited by children; passable over sockets | A handle to a store, channel, or tool in the namespace; a capability | Sketched |
| Filesystem namespace | Hierarchical names for everything | Per-agent namespace of tools, stores, channels, journals | Sketched (R15) |
| Pipes, FIFOs, sockets | Byte streams between processes, anonymous or named | IPC channels: pipelines, named peer channels, RPC to parent | Sketched (R13) |
| Signals | Asynchronous notifications (terminate, interrupt, child exited) | `SIGNAL`; interrupts delivered at instruction boundaries | v0.3 core |
| Users, groups, permission bits | Who may do what to which resource | Capabilities and authority levels | Sketched |
| Environment variables | Key–value context inherited by children | Scoped variables and budgets inherited by subagents | Sketched |
| Sessions, process groups, job control | Foreground and background jobs, terminal ownership, group signaling | Prime agent and its subagent tree; background agents; signaling a whole subtree | Partly: subtree signaling not specified |
| Standard streams (stdin, stdout, stderr) | Three conventional channels per process | Mission, material, deliverable, diagnostics (Quine's mapping); the journal separates them by event kind | Partly |
| Exit status | Small integer outcome for the parent | Tool result `status` and agent `return` value | v0.3 core |
| `chroot`, namespaces, cgroups | Restrict what a process can see and use | Per-agent namespaces and budgets; sandboxes | Sketched (R17) |
| `ioctl` | Device-specific escape hatch outside the standard verbs | Extensions advertised in `CAPS` | v0.2 onward |
| Cron, init, daemons | Scheduled and long-running background work | External triggers and durable timers | Gap (R34, R35) |

## Design lessons, including mistakes

**What worked**

- **Splitting `fork` from `exec`.** Between creating a child and starting its program, the parent can set up the child's world: redirect its file descriptors, change its directory, drop privileges. That gap is why shells are so composable. *For HB:* `SPAWN`'s spec is that gap made explicit (namespace, budget, plan); keep every part of a child's world configurable by the parent before it runs.
- **Descriptors as capabilities.** A process can only use files it has descriptors for, and descriptors can be passed deliberately. Capsicum (FreeBSD) later made this a full capability system. *For HB:* handles in a namespace, granted by the parent, are the right default.
- **Inheritance with narrowing.** Children inherit their parent's environment and can only be given less, never more, without an explicit privileged step. *For HB:* subagents inherit at most their parent's capabilities and budget.
- **Conventions over mechanisms.** Exit code 0 means success; stdout is data and stderr is diagnostics. Cheap conventions made thousands of tools composable. *For HB:* standard result shapes and event kinds matter as much as instructions.

**What went wrong, and was fixed later**

- **Signals.** Asynchronous signals can interrupt a program anywhere, making handlers notoriously hard to write correctly (only a short list of functions is safe to call inside one; interrupted system calls fail with `EINTR`). The modern fix is to turn signals into events read at a safe point (the self-pipe trick, Linux's `signalfd`). *For HB:* delivering interrupts only at instruction boundaries is the lesson already applied.
- **`fork` cost and semantics.** Copying a whole process to then replace it was wasteful and interacts badly with threads; `posix_spawn` and `vfork` were added. *For HB:* forking a context should be copy-on-write and cheap; spawning with a fresh plan should not require copying anything.
- **Ambient authority.** A process acts with all of its user's rights, whatever it was asked to do; the confused deputy was born here. *For HB:* no ambient authority; every action goes through a capability the agent was explicitly given.
- **Everything-is-a-file, incompletely.** Sockets, devices, and processes didn't quite fit the file model, so `ioctl` and special calls crept in. Plan 9 redid it consistently with per-process namespaces. *For HB:* design the namespace for agents, channels, and journals from the start rather than retrofitting.
- **Implementation-defined behavior.** POSIX left many behaviors "implementation-defined" or "unspecified," which preserved vendors' freedom but hurt portability. *For HB:* say UNSPECIFIED deliberately and sparingly, as RISC-V does.

## Implications for the Harness ISA

Most of POSIX's lessons are already reflected in v0.3 and the v0.4 candidates. Four are new or sharper:

1. **Subtree signaling (job control).** POSIX can signal a whole process group: cancel a pipeline with one keystroke. Agents need the same: kill or pause a subagent and everything it spawned, in one operation. *Candidate:* `SIGNAL` takes an agent and a scope (self, subtree).
2. **Inheritance only narrows.** State it as a rule: a child's capabilities and budget are a subset of its parent's at spawn, and widening requires a privileged, journaled operation. This closes the confused-deputy gap at spawn time.
3. **Conventions are part of the spec.** Standard event kinds, result shapes, and channel names (the agent equivalent of stdin, stdout, and stderr) deserve the same care as instructions, because they are what make independently written programs composable.
4. **No ambient authority.** Every operation names the capability it uses. This is the single most important lesson the OS world learned the hard way, and the cheapest one to build in from the start.

And one point of framing: POSIX succeeded as a *standard* because several implementations already existed and it codified their common core. A harness standard would likely follow the same path: implementations first, then a codified common core. That argues for building the reference implementation before polishing the spec further.

## Sources

- [Ke, Quine: Realizing LLM Agents as Native POSIX Processes (2026)](https://arxiv.org/html/2603.18030v1): the POSIX-to-agent mapping, including McIlroy's quote
- [Survey of Computing Principles](https://claude.ai/artifact/RKR5dXMefFaRDcQxMEVFBa): the confused deputy and capability scoping
- From general knowledge, not fetched this session: IEEE Std 1003.1 (POSIX); Ritchie & Thompson, "The UNIX Time-Sharing System" (1974); Pike et al., Plan 9 from Bell Labs (1995); Watson et al., Capsicum (2010); Gabriel, "Worse Is Better" (1989); Linux `signalfd` and the self-pipe trick; `posix_spawn`.
