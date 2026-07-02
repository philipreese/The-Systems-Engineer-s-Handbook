# Design Principles

These are the core axioms this handbook does not contradict. Every chapter's recommendations should be consistent with these principles. If a recommendation appears to conflict with one, the chapter should explain why that principle yields to a stronger constraint in that specific context.

These principles are not rules to follow blindly — they are the result of hard-won industry experience and represent the default position before specific context changes anything.

---

*Confirmed from Part I (Systems Thinking). New principles are added only when a later part
genuinely discovers one — not for every chapter's takeaway. The principles emerge from the
content, not the other way around.*

---

## Principles (confirmed through Part VII)

1. **Complexity is the primary enemy.** Almost every engineering failure traces back to unmanaged complexity. Simpler systems fail less, are easier to debug, and survive longer.

2. **Name the real trade-off.** A recommendation that hides its costs is not a recommendation — it is a sales pitch. Every "yes" has a "no" attached to it.

3. **Optimize for the engineer reading this in two years.** The author of code is rarely the most important audience. Clarity for future readers is worth more than cleverness for the current author.

4. **Reversibility is worth paying for.** Decisions that can be changed cheaply are worth making quickly. Decisions that are hard to reverse deserve more deliberation.

5. **Local optimization can damage the global system.** A change that makes one component faster, cleaner, or simpler can make the whole system slower, messier, or more fragile.

6. **Process only when it adds more than it costs.** Every process is overhead. Overhead is only justified when the benefit exceeds the cost — and that threshold is higher than most teams assume.

7. **Dependencies must point from volatile toward stable, never the reverse.** *(Discovered in Part II — [Ch 12](part02-software-architecture/ch12-dependency-direction-inversion.md), [Ch 11](part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md), [Ch 13](part02-software-architecture/ch13-coupling-cohesion-architecture-level.md), [Ch 17](part02-software-architecture/ch17-sync-vs-async-communication.md), [Ch 18](part02-software-architecture/ch18-data-ownership-boundaries.md).)* Every coupling decision — a layer, an owned interface, a bounded context, a synchronous call, a database boundary — either protects the stable side from the volatile side or fails to. The question to ask is never just "is this coupled," but "which way does the coupling point, and does that match what's actually likely to change."

8. **Mechanical enforcement is more reliable than human discipline.** *(Discovered in Part IV — [Ch 27](part04-code-organization/ch27-file-and-module-structure.md), [Ch 32](part04-code-organization/ch32-error-handling-typed-errors-vs-exceptions-vs-result-types.md), [Ch 33](part04-code-organization/ch33-when-to-write-unsafe-or-low-level-code.md).)* Any invariant the compiler, type system, or module system can verify automatically should be enforced there — not through documentation, convention, or code review. Package boundaries that enforce dependency direction, type signatures that make failure handling compiler-checked, and explicit `unsafe` regions that flag suspended guarantees are all expressions of the same principle. When mechanical verification is suspended or unavailable, the missing enforcement must be compensated through explicitly elevated human process: documented invariants, narrowed ownership, stricter review, and broader test coverage.

9. **A proxy metric, once made a mandatory target, stops measuring what it was designed to measure.** *(Discovered in Part V — [Ch 39](part05-testing-strategy/ch39-when-not-to-test.md), [Ch 41](part05-testing-strategy/ch41-coverage-what-it-measures-and-what-it-doesnt.md).)* Coverage percentages, test counts, and other automatable stand-ins for a genuinely hard-to-measure quality — confidence, correctness, code health — are useful as diagnostic floors: they reliably flag total neglect. The moment one becomes a blocking gate, it gets satisfied by whatever is cheapest to produce, which is rarely the underlying quality it was meant to approximate. The number improves; the thing it was supposed to represent does not move with it. This is not a testing-specific quirk — it is what happens to any proxy the moment measurement becomes the target instead of the diagnostic.

10. **The cost of correcting a mistake grows with the distance between when it is made and when it is discovered.** *(Discovered in Part VI — [Ch 44](part06-engineering-process/ch44-milestone-and-phase-planning.md), [Ch 46](part06-engineering-process/ch46-spec-first-development.md), [Ch 47](part06-engineering-process/ch47-code-review.md), [Ch 48](part06-engineering-process/ch48-technical-debt.md).)* A design flaw caught in a one-page spec costs minutes to fix; the same flaw caught after implementation costs days of rework, at exactly the moment redesign is most expensive and most demoralizing to ask for. A milestone plan that defers integration to a dedicated final phase pays for every hidden assumption at once, at the point in the schedule with the least slack left to absorb it. A large, slowly reviewed pull request lets a design mistake ship before anyone evaluates it with real scrutiny. Technical debt's "interest" is this same principle compounding on a longer clock: the identical shortcut costs more to correct with every week it goes unaddressed. None of this argues that mistakes are avoidable — it argues for where in a process the cheapest opportunity to catch one actually sits, and it is the throughline connecting vertical-slice planning, spec-first review, small fast-reviewed PRs, and tracked technical debt: each is a different mechanism for moving discovery earlier rather than later.
