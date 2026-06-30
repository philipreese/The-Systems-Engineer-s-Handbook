# Design Principles

These are the core axioms this handbook does not contradict. Every chapter's recommendations should be consistent with these principles. If a recommendation appears to conflict with one, the chapter should explain why that principle yields to a stronger constraint in that specific context.

These principles are not rules to follow blindly — they are the result of hard-won industry experience and represent the default position before specific context changes anything.

---

*Principles will be established as Part I (Systems Thinking) is completed. The principles emerge from the content, not the other way around.*

---

## Candidate Principles (to be confirmed or revised during Part I)

The following are working hypotheses:

1. **Complexity is the primary enemy.** Almost every engineering failure traces back to unmanaged complexity. Simpler systems fail less, are easier to debug, and survive longer.

2. **Name the real trade-off.** A recommendation that hides its costs is not a recommendation — it is a sales pitch. Every "yes" has a "no" attached to it.

3. **Optimize for the engineer reading this in two years.** The author of code is rarely the most important audience. Clarity for future readers is worth more than cleverness for the current author.

4. **Reversibility is worth paying for.** Decisions that can be changed cheaply are worth making quickly. Decisions that are hard to reverse deserve more deliberation.

5. **Local optimization can damage the global system.** A change that makes one component faster, cleaner, or simpler can make the whole system slower, messier, or more fragile.

6. **Process only when it adds more than it costs.** Every process is overhead. Overhead is only justified when the benefit exceeds the cost — and that threshold is higher than most teams assume.
