# Design Principles

These are the core axioms this handbook does not contradict. Every chapter's recommendations should be consistent with these principles. If a recommendation appears to conflict with one, the chapter should explain why that principle yields to a stronger constraint in that specific context.

These principles are not rules to follow blindly — they are the result of hard-won industry experience and represent the default position before specific context changes anything.

---

*Confirmed from Part I (Systems Thinking). New principles are added only when a later part
genuinely discovers one — not for every chapter's takeaway. The principles emerge from the
content, not the other way around.*

---

## Principles (confirmed through Part II)

1. **Complexity is the primary enemy.** Almost every engineering failure traces back to unmanaged complexity. Simpler systems fail less, are easier to debug, and survive longer.

2. **Name the real trade-off.** A recommendation that hides its costs is not a recommendation — it is a sales pitch. Every "yes" has a "no" attached to it.

3. **Optimize for the engineer reading this in two years.** The author of code is rarely the most important audience. Clarity for future readers is worth more than cleverness for the current author.

4. **Reversibility is worth paying for.** Decisions that can be changed cheaply are worth making quickly. Decisions that are hard to reverse deserve more deliberation.

5. **Local optimization can damage the global system.** A change that makes one component faster, cleaner, or simpler can make the whole system slower, messier, or more fragile.

6. **Process only when it adds more than it costs.** Every process is overhead. Overhead is only justified when the benefit exceeds the cost — and that threshold is higher than most teams assume.

7. **Dependencies must point from volatile toward stable, never the reverse.** *(Discovered in Part II — [Ch 12](part02-software-architecture/ch12-dependency-direction-inversion.md), [Ch 11](part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md), [Ch 13](part02-software-architecture/ch13-coupling-cohesion-architecture-level.md), [Ch 17](part02-software-architecture/ch17-sync-vs-async-communication.md), [Ch 18](part02-software-architecture/ch18-data-ownership-boundaries.md).)* Every coupling decision — a layer, an owned interface, a bounded context, a synchronous call, a database boundary — either protects the stable side from the volatile side or fails to. The question to ask is never just "is this coupled," but "which way does the coupling point, and does that match what's actually likely to change."
