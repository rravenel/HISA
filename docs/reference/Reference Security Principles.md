# Reference: Security Principles

Oct 20, 2026 · @Rene Ravenel

What fifty years of security engineering says about agents that read untrusted text and hold real authority, and what the Harness ISA should build in.

## The threat: agents are a new kind of deputy

In 1988 Norm Hardy described a compiler on a shared system that held legitimate rights to write a billing file. A user asked it to write debug output to that file's path, and it obliged, overwriting the billing records. The compiler wasn't compromised; it was confused about *whose* authority it was exercising. Hardy called it the **confused deputy**.

An agent that reads untrusted text while holding tools is the same structure with three aggravations:

- **Instructions and data share one channel.** A model processes the user's request and a web page's contents in the same context and cannot reliably tell which is which. There is no syntactic boundary to enforce.
- **Agents act across many systems in one session,** so one confused decision can move data between security domains.
- **Attackers adapt.** Text is free to iterate on, and published defenses that rely on detecting injections have repeatedly fallen to adaptive attacks. The UK's NCSC has warned that prompt injection may never be fully fixed at the input layer.

The consequence for design is the lesson the OS world learned decades ago: **don't make the deputy read more carefully; give it less to be confused with.** Security has to come from structure (what each part of the system *can* do) rather than from the model's judgment about what it *should* do. Detection still helps, but only as a layer on top.

## Classic design principles, applied

Saltzer and Schroeder's 1975 paper *The Protection of Information in Computer Systems* listed eight design principles and two more they considered imperfectly applicable. All ten still hold, and each has a direct reading for harnesses.

| Principle | Meaning | Harness application |
| --- | --- | --- |
| Economy of mechanism | Keep the protection design small and simple | A small base ISA; one mediation point (the environment) rather than checks scattered through programs |
| Fail-safe defaults | Base access on permission, not exclusion; default is no access | The default subagent sees only its own tools and `return`; everything else is granted |
| Complete mediation | Check every access to every object | Every instruction, from the model or from code, passes through the environment's capability check; no back doors for "trusted" programs |
| Open design | Security must not depend on secrecy of the design | Publish the ISA and its guarantees; secrets live in credentials, not in hidden prompts |
| Separation of privilege | Require two keys where feasible | Promotion into harness positions needs tests *and* human approval; changes to the authoring agent need review |
| Least privilege | Every program operates with the least privilege needed | Per-agent namespaces; inheritance only narrows; per-task capabilities rather than per-user |
| Least common mechanism | Minimize mechanism shared between users | Separate sandboxes and sessions per agent; shared stores only where coordination needs them |
| Psychological acceptability | Protection must be easy to use correctly | Approval prompts that are rare and meaningful, so people don't click through them |
| Work factor | Compare the cost of circumvention with the attacker's resources | Assume attacker text is free to iterate; never rely on the model resisting it |
| Compromise recording | Reliably record that a compromise happened | The journal: every action, verdict, and transform is recorded with provenance, so incidents can be reconstructed |

## Capability security

The **object-capability model** goes back to Dennis and Van Horn (1966) and was developed most fully in Mark Miller's work (*Robust Composition*, 2006). Its rules are simple:

- A **capability** is an unforgeable reference that both *designates* a resource and *authorizes* its use. There is no separate permission lookup.
- You can only use what you hold, and you can only get capabilities by creation, by being given one, or from something you already hold.
- There is **no ambient authority**: nothing is available just because of who you are or where you run.

The guiding principle is **POLA, the principle of least authority**: not just minimal permissions per user, but minimal authority per component, recursively down to each object. Because a capability bundles designation with authority, the confused deputy problem largely disappears: the compiler in Hardy's story would have written only to files whose capabilities the user handed it.

Three patterns from the capability literature matter for agents:

- **Attenuation.** Wrap a capability to grant less: read-only access to a store, a mail tool that can only send to one address, a budget-capped model role.
- **Revocation.** Grant through a forwarder you can later cut, so authority given to a subagent can be withdrawn.
- **Membranes.** Wrap everything that crosses a boundary so the whole subgraph of authority can be attenuated or revoked at once, for example everything a subagent ever received.

*Harness reading:* tools, stores, channels, and model roles in an agent's namespace should be capabilities in this exact sense. A parent hands a subagent attenuated capabilities at `SPAWN`; the subagent can pass them on only in narrower form; and the parent can revoke the whole set by revoking the membrane.

## Structural defenses built for agents

Since 2023 the field has converged on structural rules and architectures rather than detection. Five are worth knowing.

**The lethal trifecta** (Simon Willison, June 2025). An agent that combines access to private data, exposure to untrusted content, and the ability to communicate externally can be tricked into exfiltrating that data. Willison's advice is to avoid the combination entirely.

**The Agents Rule of Two** (Meta, October 31, 2025). Within a session, an agent should have at most two of: \[A\] processing untrustworthy inputs, \[B\] access to sensitive systems or private data, \[C\] changing state or communicating externally. Adding "changing state" covers harms beyond exfiltration. When a task needs all three, split it across sessions so no single one holds all three, or require human approval for the third. It descends from Chromium's "Rule of 2" for browser code. A noted limit: it constrains single agents, and a chain of individually compliant agents can still combine all three.

**The dual-LLM pattern.** A privileged model plans and calls tools but never sees untrusted content; a quarantined model reads untrusted content but has no tools. The privileged side refers to quarantined outputs by symbolic name rather than reading them.

**CaMeL** (Google DeepMind, March 2025, "Defeating Prompt Injections by Design"). It extracts the control flow and data flow from the trusted user query, so untrusted data retrieved later cannot change what the program does. The privileged model writes the plan as code in a restricted Python; a custom interpreter runs it, tracks provenance, and attaches **capabilities** (metadata on who may read a value and where it came from) to every value; policies check those capabilities before each tool call. On the AgentDojo benchmark it solved 77% of tasks with provable security versus 84% for an undefended agent: a modest utility cost for a structural guarantee.

**Information-flow control: FIDES** (Microsoft Research, 2025). A planner that tracks **confidentiality and integrity labels** on every message, tool call, and result. When a model combines inputs, its output takes the most restrictive combination of their labels, and a policy engine deterministically enforces rules at each action. FIDES adds primitives that hide labeled data from the planner and use an isolated model with structured output to extract only typed facts from it, broadening the range of tasks that can be done securely.

The common thread: **separate control from data, label everything, and enforce policy deterministically outside the model.**

## Implications for the Harness ISA

The Harness ISA is unusually well placed for structural security, because the journal, plans, and the single mediation point already exist. Most of the work is making existing ideas precise.

1. **Authority labels become information-flow labels (sharpens R4).** Every journal event carries a confidentiality label and an integrity label. Combining inputs (an `INFER` over a view, a summary, a transform) yields the most restrictive combination. Policies check labels at each `CALL`. This is FIDES's model applied at the ISA level, so every harness gets it rather than each re-implementing it.
2. **Control flow from trusted sources only.** Plans and programs (the control flow) are written from trusted inputs; untrusted content enters only as data. This is CaMeL's core move, and HB's split between plans (control) and views (data) makes it natural. A view built from low-integrity content cannot authorize a high-consequence call.
3. **Quarantined inference as a standard pattern.** An `INFER` with an empty namespace and a required output schema is a quarantined model: it can read anything and do nothing, returning only typed data. Make it a named, first-class mode so programs use it by default for untrusted content.
4. **The Rule of Two as a checkable property.** Because an agent's namespace is explicit, the environment can compute at `SPAWN` time whether the agent holds untrusted inputs, sensitive data, and state-changing or outbound capabilities together, and refuse or require approval. Across agents, the check must follow channels too, since chains of compliant agents can still combine all three.
5. **Capabilities in the full sense.** Namespace entries are unforgeable handles that designate and authorize at once; attenuation, revocation, and membranes are standard operations; inheritance only narrows; there is no ambient authority.
6. **Secrets as capabilities (R36)**, never values in views or the journal.
7. **Separation of privilege for self-modification.** Promotion into harness positions, and any change to the authoring agent, requires two keys: passing verification *and* human approval. Guardrails and policy code sit outside the reach of the agent they constrain.
8. **Inter-agent messages are untrusted by default.** A message from another agent carries that agent's integrity label; peer channels are an injection path, not a trusted bus.

Proposed v0.4 candidates: R47 confidentiality and integrity labels with deterministic propagation; R48 quarantined inference mode; R49 Rule-of-Two check at spawn and across channels; R50 attenuation, revocation, and membranes as standard capability operations.

## Sources

- [Saltzer & Schroeder design principles (summary with all ten)](https://jeremyjordan.me/security-design) and [Wikipedia overview](https://en.wikipedia.org/wiki/The_Protection_of_Information_in_Computer_Systems)
- [Prompt injection as a confused deputy](https://tianpan.co/blog/2026-05-17-prompt-injection-confused-deputy-capability-scoping) and [What is the confused deputy problem](https://decagon.ai/glossary/what-is-the-confused-deputy-problem)
- [Miller, Yee & Shapiro, Paradigm Regained: Abstraction Mechanisms for Access Control](https://pdos.csail.mit.edu/6.1810/2004/readings/miller03paradigm.pdf) and [Object-capability model](https://en.wikipedia.org/wiki/Object-capability_model)
- [Willison, The lethal trifecta for AI agents (June 2025)](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)
- [Willison on Meta's Agents Rule of Two](https://simonw.substack.com/p/new-prompt-injection-papers-agents) and [a critique of its single-agent scope](https://kenhuangus.substack.com/p/the-rule-of-two-vs-reality-why-metas)
- [Debenedetti et al., Defeating Prompt Injections by Design (CaMeL)](https://sotaverified.org/papers/defeating-prompt-injections-by-design) and [Willison's explainer](https://simonwillison.net/2025/Apr/11/camel/)
- [Costa et al., Securing AI Agents with Information-Flow Control (FIDES)](https://www.microsoft.com/en-us/research/publication/securing-ai-agents-with-information-flow-control/)
