# Ch 30 — Comments: What to Comment, What Not To

**Prerequisites:** [Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Naming Conventions and When They Matter](ch28-naming-conventions-and-when-they-matter.md), [API Surface Design: What to Expose, What to Hide](../part02-software-architecture/ch15-api-surface-design-expose-hide.md)

**New vocabulary introduced:** comment rot

**Key takeaways:**
- A comment is a maintenance liability: it must stay synchronized with code that changes, but unlike code, nothing enforces that it does.
- The central test is WHY vs. WHAT. A comment restating what the next line already says is pure cost. A comment capturing a hidden constraint, a lock-ordering requirement, or a workaround for an external bug preserves knowledge the code cannot express.
- Comment rot — a comment that no longer matches the code — is worse than no comment. It actively misleads a reader who has no way to know it's stale.
- Implementation comments (for the next maintainer) and API documentation comments (for the next caller) serve different audiences and carry different obligations.
- If the urge to comment comes from a block of code that needs explanation, the right response is usually to extract the block into a well-named function, not to add prose.

---

Repository-level design rationale — the reasoning behind large-scale decisions — belongs in Architecture Decision Records, covered in [Part VI, Ch 45](../part06-engineering-process/ch45-architecture-decision-records.md). API documentation as a publishing discipline is covered in [Part VIII, Ch 64–68](../part08-documentation/). This chapter is about inline code comments: what they're for, when they earn their maintenance cost, and when they don't.

---

### Comment the Why, Never the What

**What it is:** The central test for whether an inline comment is justified — whether it records information that cannot be recovered by reading the code itself.

**Why it exists:** Source code already describes behavior mechanically. A comment that translates the next line into English doubles the maintenance burden with no information gain. Only a comment can explain *why* the code does what it does: the external constraint that forced the choice, the invariant the code depends on but doesn't visibly enforce, the behavior that would surprise a careful reader who had no prior context.

**Options:**

1. **Capture context** — document a hidden constraint, a lock-ordering requirement, a hardware quirk, a known bug in a third-party API, a non-obvious invariant the implementation relies on.
2. **Restate mechanics** — describe the operations the code already makes visible.
3. **No comment** — let the implementation speak for itself.

**Trade-offs:**

[Strong Recommendation] **Capture context** when the information cannot be recovered from the code. The invisible constraints a comment preserves are exactly the constraints a future maintainer will accidentally violate without it. The cost is real: the comment must be deleted or updated when the constraint changes.

**Restate mechanics** has no upside for a competent reader, and a concrete downside: it will become wrong the moment the code changes. `// increment index by 1` above `index++` contributes nothing and will never be updated when the index step changes to 2.

**No comment** is often correct. Well-named functions and expressive types communicate more than prose can, without the maintenance debt.

**The categories of comments that earn their cost:**
- A hidden constraint from outside the codebase: *why* a specific ordering is required, *why* a sleep is intentional, *why* a branch handles a case that looks impossible
- A workaround for a specific external bug: version, ticket reference, what breaks if removed
- A lock-ordering rule that the type system cannot express
- A non-obvious invariant the surrounding code depends on but doesn't enforce

**Common failure modes:** *The Stale Lie.* An engineer writes a comment explaining exactly what a complex block of code does. Six months later, a bug fix alters the behavior. The engineer updates the code but ignores the comment. The next maintainer reads the comment, trusts it, and wastes days debugging a reality that contradicts the prose. A stale comment is not neutral: it is an active misdirection.

**Example:** The Linux kernel enforces this discipline explicitly. Kernel developers are instructed not to explain what or how the code operates. Inline comments are reserved for what cannot be deduced from the C alone: memory barrier requirements, processor errata workarounds, hardware timing constraints, lock-ordering rules that exist because of a specific race condition documented elsewhere. The comments explain the physical universe the code operates in, not the code itself.

---

### Treat Every Comment as a Maintenance Liability

**What it is:** A comment is not free. It is a second description of the system that must evolve alongside the first one — the code — with no compiler to check that it has.

**Why it exists:** Software evolves; comments frequently do not. Unlike the implementation, a comment that no longer matches the code produces no warning, no test failure, and no compile error. It just silently misleads every future reader who encounters it.

**Comment rot** is the term for this failure: a comment that no longer matches the underlying code. It is strictly worse than no comment at all. A reader with no comment must reason from the code; a reader with a stale comment reasons from a wrong model and may never discover the divergence.

**When to choose each option:**

[Consensus] Keep comments minimal. Write them only when the benefit clearly exceeds the ongoing maintenance cost. The comment that prevents a regression is worth writing. The comment that restates the function name is not. When a comment no longer communicates unique information, removing it is preferable to updating it — the survivor population of comments should be ones a reader can trust.

**Common failure modes:** A production incident is extended by hours because a comment in the code described caching behavior that was changed two years earlier. Every engineer who reads the comment assumes it is current. The actual behavior is the opposite. The misleading comment is more dangerous than silence would have been, because silence would have forced engineers to read the code.

---

### Extract Code Before Adding a Comment

**What it is:** When the urge to write a comment comes from a block of code that needs explanation, the correct first question is whether the block should become a well-named function instead.

**Why it exists:** A comment describes behavior in prose that lives outside the executable path. A function name *becomes* the behavior's description at every call site, enforced by the compiler. As established in [Ch 28](ch28-naming-conventions-and-when-they-matter.md), naming is a native abstraction; commenting is not. When the intent can be expressed directly in the structure of the code, that expression is more durable than prose that will drift.

**Options:**

1. **Extract to function** — move the block into a function whose name is the description the comment would have carried.
2. **Comment the block** — leave the code inline and add prose above it.

**Trade-offs:**

[Strong Recommendation] **Extract to function** when the comment would explain *what* the block does. The function name eliminates the comment's maintenance burden entirely — the compiler now enforces the name. The block also becomes independently testable.

**Comment the block** only when extraction would damage performance or conceptual cohesion: a tight mathematical kernel where splitting across function call boundaries would either introduce unacceptable overhead or scatter the algorithm across pieces that only make sense together.

**Common failure modes:** *The Header Comment Monolith.* A 300-line function is visually divided into sections:

```
// --- FETCH DATA ---
// --- PARSE DATA ---
// --- SAVE DATA ---
```

Each section header is an unbuilt function. The comments are doing the organizational work that function extraction was designed to do — and doing it in a way that doesn't compose, doesn't test independently, and will drift from the code beneath it.

**Example:** In the Clean Code tradition, a comment explaining a block is treated as a code smell that indicates a missed extraction. The conditional `if (employee.flags & HOURLY_FLAG && employee.age > 65)` demands a comment. Extracted to `if (employee.isEligibleForPension())`, the comment is gone, the intent is readable at the call site, and the eligibility rule has a name that the entire codebase can reference.

---

### Distinguish Implementation Comments from API Documentation

**What it is:** Implementation comments and API documentation comments are written for different audiences and carry different obligations. Treating them as the same category produces documentation that serves neither audience well.

**Why it exists:** A maintainer reading the function body needs to understand *why* an internal decision was made. A caller using a public interface needs to understand *what* the contract is — inputs, outputs, side effects, failure modes — without reading the implementation. Conflating the two produces docstrings that leak internal details to callers, and inline comments that carry contract information invisible to IDE tooling.

**Options:**

1. **Implementation comments** — inline, inside the function body, aimed at the next person who modifies the code.
2. **API documentation comments (docstrings)** — attached to public declarations, parsed by tooling to generate IDE tooltips and documentation sites, aimed at the next person who *calls* the code.

**Trade-offs:**

[Strong Recommendation] **API documentation comments are part of the interface contract**, not optional commentary. A public function without a docstring forces every caller to read the implementation. A docstring that leaks internal implementation details couples callers to private decisions — when the internal mechanism changes, the public contract changes with it even though nothing externally visible has shifted.

**Common failure modes:** *The Leaky Docstring.* A developer documents a public interface with internal details: `"""Fetches the user by scanning the Redis cache array."""` When the cache moves from Redis to Memcached, the docstring is wrong, and every caller who relied on it for mental model now holds a broken model. The docstring should have said *what* the function does for the caller, not *how* it currently does it.

**Example:** Python's PEP 257 codifies the distinction structurally. An inline comment using `#` is discarded by the parser. A docstring using `"""..."""` immediately below a declaration is not discarded — it becomes the `__doc__` attribute on the function or class object at runtime, accessible to IDE tools, `help()`, and documentation generators. Python makes the two categories physically different artifacts because their obligations are different.

---

### Why Smart Engineers Disagree: The Self-Documenting Code Absolutists

A persistent position in software engineering holds that any comment is a failure of expressiveness. If the code needs a comment, the argument goes, the names are wrong, the functions are too long, or the abstractions are bad. Some teams translate this into a policy of banning all comments from the repository.

The position is right about what-comments. A comment that restates what the next line does is always avoidable. It should be eliminated, not by better commenting, but by better code.

The position fails against external constraints. A bug in a third-party API is not self-documenting. A workaround for a race condition in a deprecated network switch cannot be expressed through clever variable names. A memory barrier that must execute before a specific store, because of a processor erratum documented in a hardware manual, cannot be inferred from the surrounding C. Banning comments in these cases doesn't make the code cleaner — it erases the system's institutional memory.

The pragmatic position: write code that is structurally clear enough to make all mechanical comments unnecessary. Reserve comments for what the compiler cannot see — the external constraints, the invisible invariants, the decisions that would look wrong without context and right with it. That population of comments should be small, should be trusted, and should be actively pruned when the constraint it documents no longer applies.
