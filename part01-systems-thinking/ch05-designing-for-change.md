# Chapter 5 — Designing for Change

**Prerequisites:** [Ch 02 — Complexity Is the Enemy](ch02-complexity-is-the-enemy.md), [Ch 03 — Coupling and Cohesion](ch03-coupling-and-cohesion.md), [Ch 04 — Abstraction and Information Hiding](ch04-abstraction-and-information-hiding.md). Specifically: accidental complexity, afferent coupling, and information hiding as a tool for isolating volatile decisions.

**New vocabulary introduced:** Open/Closed Principle (OCP), axis of variation, future-proofing

**Key takeaways:**
- Most systems fail not because they were badly designed, but because they made the wrong things hard to change. Designing for change is constraint selection: deciding which dimensions of a system are allowed to move and which are explicitly locked.
- Stability and flexibility are not opposites to balance uniformly — they are orthogonal concerns to assign deliberately. The standard shape is a rigid interface over a fluid implementation: freeze the contract, let everything behind it churn.
- The Open/Closed Principle is a useful lens for managing regression risk, not a mandate for indirection everywhere. Applied where change is genuinely additive, it isolates risk. Applied where requirements actually contradict prior rules, it adds abstraction with no benefit.
- Designing for change means identifying specific, known axes of variation and making those cheap. Future-proofing means paying complexity now for unnamed, speculative requirements that may never arrive. The first is a targeted bet; the second is usually a tax with no return.

---

## Purpose

A system that cannot change is already obsolete the day it ships. A system where everything can change at once is in a permanent state of low-grade collapse — every modification risks breaking every dependent. This chapter is about the deliberate partition between the two: what should remain fixed, and what should be free to evolve cheaply.

This is not future-proofing. Future-proofing speculates about unspecified requirements and pays complexity now for benefits that may never materialize. Designing for change is narrower and more disciplined: name the specific axis a system is likely to vary along, and make movement along that axis cheap. Everything else stays simple, even if that means it is harder to change later — because most of "everything else" will never need to change at all.

---

## What Should Be Stable vs. What Should Be Flexible

**What it is:** System design is fundamentally about choosing invariants. Some parts of a system should not change often — interfaces, contracts, data shapes. Other parts should be designed to change frequently — implementations, algorithms, internal data structures. A well-designed system stabilizes the boundary so the interior can be modified aggressively without alerting anything outside it.

**Why it exists:** If a core component changes its contract, every component with high afferent coupling to it (Ch 03) must also change — a cascading, coordinated failure. Stabilizing the boundary converts a coordination problem into a local one: the interior can be rewritten by one team, on one schedule, with no one else needing to know.

**Options:**
1. **Rigid interface, fluid implementation** — the public contract is frozen; algorithms, data structures, and dependencies behind it can be rewritten entirely
2. **Fluid interface, rigid implementation** — the external contract evolves rapidly to match immediate needs, while the execution engine behind it stays a static monolith
3. **Everything stable / everything flexible** — the two degenerate cases, useful only as a reminder that neither extreme is viable at scale

**Trade-offs:**

| Profile | Protects | Costs |
|---|---|---|
| Rigid interface / fluid implementation | Downstream consumers; ecosystem trust | Locks the system out of changes that genuinely require a new contract shape |
| Fluid interface / rigid implementation | Rapid alignment with immediate caller needs | Breaks every consumer on each change; forces lockstep deployment |
| Everything stable | Predictability | Resistant to evolution the business actually needs |
| Everything flexible | Nothing, durably | Degrades into unpredictability and hidden coupling as nothing can be relied upon |

**When to choose each:**
- *Rigid interface / fluid implementation:* public APIs, operating system abstractions, foundational databases, anything with high afferent coupling.
- *Fluid interface / rigid implementation:* internal data-transformation layers, rapid prototypes, edge components with zero afferent dependents.
- *Fully flexible:* prototypes and experimental systems where the correct shape isn't known yet — this is a starting state, not a destination.

**Common failure modes:**
- **The "God Interface":** an attempt to stabilize a boundary fails because the interface never captured a genuinely stable domain concept — it's a dumping ground of unrelated methods that must change weekly, breaking downstream consumers despite the appearance of a contract.
- Locking in an API shape before the domain is understood, then being unable to evolve it without breaking every client.
- Over-stabilizing internal logic that should have stayed cheap to change, inflating the cost of ordinary feature work.

**Example:** The Linux kernel enforces a strict rigid-interface boundary at the syscall ABI. Linus Torvalds's governing rule is blunt: "Never break user space." Internal kernel data structures, the scheduler, and memory management are rewritten constantly — entire subsystems have been replaced more than once. Yet a binary compiled for Linux in 1998 will, with very few exceptions, still run on a modern kernel. The stability is not an accident of slow development; it is a deliberate commitment that buys the kernel team the freedom to change everything behind it. **[Strong Recommendation: when a component has high afferent coupling, stabilize the contract explicitly and treat that decision as a commitment, not a default]**

---

## The Open/Closed Principle — Useful but Incomplete

**What it is:** Software entities should be open for extension but closed for modification: new behavior is added through new code, not by editing and re-risking existing, tested code. It is a pragmatic lens for managing regression risk, not a general law that applies uniformly.

**Why it exists:** Modifying heavily tested, production-hardened code carries real regression risk. Extending a system by adding new code — a plugin, a new implementation of an existing interface — isolates the risk of the new feature from the stability of the old ones.

**Options:**
1. **Strict OCP** — behavior is altered exclusively through new classes/modules conforming to existing interfaces (plugins, dependency injection, polymorphism)
2. **Partial OCP** — a stable core with a small number of deliberate, controlled extension points
3. **Pragmatic modification** — directly editing existing code, including its conditional logic, to accommodate new rules

**Trade-offs:**
- *Strict OCP:* never breaks existing test suites and is safe to deploy incrementally, but introduces real indirection cost — following execution flow means jumping through interfaces and factories, and that cost is paid by every future reader, not just the one who built the extension point.
- *Partial OCP:* balances flexibility and simplicity, but only works if the extension points were drawn in the right place — drawing them in the wrong place is itself a form of premature abstraction (Ch 04).
- *Pragmatic modification:* keeps code volume low and the execution path obvious, but risks regressions in shared logic and invalidates the safety the principle was meant to provide.

**When to choose each:**
- *Strict OCP:* frameworks, plugin systems, and infrastructural libraries where the axis of change is strictly additive — a new serializer, a new payment provider, a new SQL dialect.
- *Partial OCP:* most backend services and database-adjacent code, where a stable core needs a small number of well-understood extension points.
- *Pragmatic modification:* core business logic, where new requirements frequently contradict or replace old rules rather than extending them — additive extension is not possible when the new rule and the old rule cannot coexist.

**Common failure modes:**
- **Premature extension hooks:** an engineer builds a plugin architecture and abstract interfaces for use cases that don't exist yet. When the actual requirement arrives, the system is too complex to modify — violating the very principle it was trying to uphold.
- Over-engineered plugin systems that never gain a second plugin.
- Deep inheritance hierarchies that are unreasoned-about under real change pressure, because OCP was applied as a default rather than a deliberate choice.

**Example:** PostgreSQL's wire protocol is practically closed for modification — its binary layout has been stable across decades of major versions, protecting every driver in the ecosystem. But the engine itself is famously open for extension: PostGIS adds an entire geospatial type system and indexing strategy without modifying PostgreSQL's core source. The protocol is closed; the engine is open. These are different boundaries, stabilized for different reasons. **[Consensus: apply OCP where the axis of change is additive; abandon it where new requirements genuinely contradict old ones, and modify the code directly]**

---

## Designing for Change vs. Future-Proofing

**What it is:** Designing for change means identifying a known, specific axis of variation and making movement along it cheap. Future-proofing means anticipating unknown future requirements and building generality upfront to absorb them. The first is grounded in observed or strategically committed variability. The second is speculation.

**Why it exists:** Business requirements change, but not randomly — a systems engineer's job is to predict *how* a specific, known domain will evolve. Predicting *whether* an entirely unrelated domain will materialize is guessing, and guessing wrong is expensive in a specific way: the complexity is paid immediately, and the benefit may never arrive.

**Options:**
1. **Axis-based (targeted) design** — explicit variability points for a named, specific dimension of change
2. **General-purpose design** — broad, abstract interfaces meant to absorb any future requirement
3. **Minimal design** — no anticipation of change at all; pay the refactor cost later if and when change actually arrives

**Trade-offs:**
- *Axis-based design:* reduces change cost for the variation it targets, but provides no benefit for variation it didn't anticipate — and that's an acceptable trade, because most unanticipated variation never happens.
- *General-purpose design:* offers theoretical flexibility for anything, but the accidental complexity is paid immediately and continuously, by every engineer who has to navigate generic, parameterized code paths to do ordinary work.
- *Minimal design:* fastest initially, defers all change cost to whenever change actually arrives — which is often cheaper than it looks, because real change rarely matches what speculative generality guessed.

**When to choose each:**
- *Axis-based:* when historical data or an explicit, funded business strategy names the dimension — multiple authentication providers, multiple tax jurisdictions for a company actively expanding internationally.
- *General-purpose:* rare; justified mainly in foundational libraries and platforms meant to be consumed by unknown future callers.
- *Minimal:* early-stage systems and experimental products, where the cost of guessing wrong about the axis exceeds the cost of refactoring later.

**Common failure modes:**
- **The EAV (Entity-Attribute-Value) anti-pattern:** an engineer "future-proofs" a relational schema with a generic `(entity_id, attribute_name, attribute_value)` table so the team "never has to run a migration again." This destroys indexing, query planning, and relational integrity, and the system collapses under read load that a normal schema would have handled trivially.
- A system built to support "multiple database backends" that only ever runs on PostgreSQL, paying for a repository-pattern indirection layer that never reduced real change cost because the second backend never arrived.
- Missing a real axis of variation — treating something as fixed (a storage backend, an auth provider) that later genuinely needs to vary, because no one named it as a candidate axis up front.

**Example:** Linus Torvalds did not future-proof Git with a generic, extensible database layer to handle "whatever we might want to store later." He targeted one specific, named need: quickly identify and retrieve immutable trees of content. The result — a content-addressable object store of blobs, trees, and commits hashed by content — has stayed mathematically stable for two decades. Because that one axis was correctly identified and rigorously stabilized, entirely new capabilities (rebasing, sparse checkouts, partial clones) were built cheaply on top of it without ever touching the object model. **[Consensus: if you cannot name the exact requirement an abstraction is for, you cannot justify its complexity cost yet]**

---

## Stable-Core Case Studies

Three systems illustrate the same underlying discipline applied at different layers: stabilize one specific, correctly identified thing, and let everything else move freely.

**Git's content-addressable storage:** what's stable is object identity (content-hash addressing), the immutability of stored objects, and the basic commit graph structure. What's flexible is ref management, transfer protocols, and every UI-level feature built on top. The complexity that would otherwise live in the core instead lives in the higher layers — indexing, refs, plumbing commands — which is exactly where it's cheapest to change.

**The Linux syscall ABI:** what's stable is syscall numbers, signatures, and behavioral contracts. What's flexible is the scheduler, memory management, and every hardware-specific implementation behind them. The cost of this stability is real — the kernel carries decades of legacy behavior it cannot remove, and that accumulation is itself a long-term complexity tax. The trade was made deliberately, in exchange for an ecosystem that trusts the interface absolutely.

**PostgreSQL's wire protocol:** what's stable is the query/response protocol structure, authentication handshake, and basic session semantics. What's flexible is the query planner, storage engine, and indexing strategy — meaning major performance improvements have shipped for decades without forcing a single driver rewrite. The recurring failure mode here is conflating protocol stability with implementation stability: a client that relies on undocumented response behavior can still break across versions, even though the documented contract never moved.

All three stabilize the contract and leave the implementation free. None of them attempt to stabilize everything, and none of them leave everything fluid.

---

## Why Smart Engineers Disagree

The fiercest disagreement in this chapter is about *when* to freeze an interface, and it tracks the same fault line as Ch 04's abstraction-altitude disagreement.

Engineers optimizing for feature velocity and team autonomy — often product engineers — resist freezing contracts early. They argue that locking an interface before the domain is understood solidifies ignorance into architecture, and prefer to let API shapes, payloads, and schemas move continuously to track shifting product requirements.

Engineers optimizing for ecosystem scale and reliability — often platform or systems engineers — enforce stable contracts aggressively. They understand that when an interface shifts, the cost is not paid by its author; it's paid by every downstream team that has to drop its roadmap to fix a broken integration.

Both are managing real risk, and both fail at the extremes. Freezing too early locks in a guess. Freezing too late destroys the trust that makes other teams willing to depend on a service at all. The resolution most mature systems converge on is sequential, not simultaneous: stay fluid while the primary use cases are still being validated, then explicitly transition the boundary from fluid to rigid once they are — permanently shifting the burden of future change from the consumer to the implementer. Git, Linux, and PostgreSQL all made this transition once, deliberately, rather than treating "stable" as a permanent default from day one.

*Concepts expanded in later chapters: API versioning strategies (Part II, Ch 16), branching strategies (Part VII, Ch 50), Architecture Decision Records (Part VI, Ch 45).*
