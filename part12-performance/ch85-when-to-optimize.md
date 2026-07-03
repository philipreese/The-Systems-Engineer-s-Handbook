# Ch 85 — When to Optimize (and When Not To)

**Prerequisites:** [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md) (cost of change vs. cost of execution, Principle 6), [Cost Models and Mechanical Sympathy](../part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md) (latency hierarchy, referenced not re-derived), [Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md) (Little's Law, Theory of Constraints), [Distributed Tracing](../part09-observability/ch72-distributed-tracing.md) (the checkout endpoint case, the trace-versus-profiler boundary), [Error Budgets and SLOs](../part09-observability/ch73-error-budgets-and-slos.md) (SLI, SLO, error budget)

**New vocabulary introduced:** Optimization Gate, measured bottleneck

**Key takeaways:**
- [Strong Recommendation] The **Optimization Gate**: performance work is justified only when a documented requirement (an SLO, a cost budget, a measured UX threshold) is actually being violated, and evidence points to a specific system-level constraint causing the violation. Absent either condition, performance work is speculative refinement, not engineering — Principle 6 (process only when it adds more than it costs) applied to performance specifically.
- A component that is slow in isolation is not necessarily a **measured bottleneck** — the thing actually limiting the system against its requirement. Ch 08 already showed that local optimization can leave the global problem untouched; this chapter turns that warning into the precondition for starting any performance work at all.
- Knuth's "premature optimization is the root of all evil" is routinely quoted stripped of its context. The actual claim is narrower and more useful: roughly 97% of a program's code has no measurable effect on its runtime cost, and the error is not optimizing — it's optimizing the wrong 97%.
- This chapter is the gate; it does not supply the method. Ch 86 covers how a bottleneck is actually found once the gate says one must be.

---

Ch 01 closed with a specific, qualified recommendation: optimize the cost of change over the cost of execution *until profiling identifies a specific runtime bottleneck*. That qualifier has done a lot of quiet work throughout this handbook — every chapter that recommended a boundary, an interface, or a layer over a faster but more rigid alternative was implicitly relying on it. This chapter is where that qualifier finally gets a full treatment: what counts as a bottleneck, and what makes "until" arrive.

It plays the same role for this Part that Ch 74 played for Part X and Ch 79 played for Part XI: every later chapter here assumes this gate has already been passed. Ch 86 through Ch 90 are about how to find a bottleneck, how to reason about it, and how to fix it — none of them re-argue whether looking was justified in the first place.

### Decision: Gate Optimization on a Documented Requirement, or Act on Perceived Slowness

**What it is:** Whether a change to a system's runtime behavior requires a real, pre-existing requirement it is failing to meet — an SLO breach, a cost ceiling, a measured conversion or retention threshold — or whether "this feels slow" or "this looks inefficient" is sufficient justification on its own.

**Why it exists:** Performance work is not free. It trades away the cost-of-change advantages Ch 01 already established as the dominant long-term cost for most systems: it adds rigidity, introduces specialized code paths, and consumes engineering time that has an opportunity cost. Without a gate, that trade gets made on intuition, and intuition about what is slow is systematically unreliable in any system with more than one interacting component — which is the entire subject of Ch 08.

**Options:**
- **Reactive, requirement-gated optimization** — performance work begins only once a documented SLO, cost budget, or UX threshold (Ch 73's vocabulary: an SLI crossing its SLO) is actually being violated or is demonstrably about to be.
- **Speculative optimization** — code paths are tuned during initial implementation or ambient refactoring based on stylistic conventions, theoretical future scale, or a developer's sense that something looks inefficient.

**Trade-offs:** Requirement-gated optimization keeps engineering effort pointed at the 3% of the system that Knuth's observation says actually matters, and every optimization it produces is traceable to a real, measurable outcome. Its cost is that a genuine architectural ceiling can go unaddressed until it is hit under real load, forcing reactive, higher-stress work at the worst possible time. Speculative optimization can pre-empt that ceiling and occasionally produces a system that is fast from day one, but it spends real engineering time against code that Knuth's 97% majority statistically will not need it, and it routinely produces exactly the kind of premature rigidity Ch 01 already warned against — an execution-optimized module that is now expensive to change for a performance requirement nobody has confirmed exists.

**When to choose each:** [Strong Recommendation] Requirement-gated optimization by default for product features, business logic, and any code path without a known, extreme physical constraint — the same population Ch 01 already recommended optimizing for change over execution. Speculative tuning is defensible only for foundational infrastructure where crossing a boundary carries a known, physically unavoidable penalty regardless of load — a storage engine's on-disk format, a lock-free data structure shared across threads, a hot serialization path already proven, by a prior system, to dominate cost at any realistic scale.

**Common failure modes:** *Fashion-driven tuning.* A team replaces a clean, readable JSON serialization call with a hand-rolled byte-manipulation routine before the service has served a single production request, saving microseconds on a path that spends 200 milliseconds waiting on a downstream database call. The change makes the code harder to modify and was never validated against any actual requirement — it was validated against the developer's aesthetic sense that manual serialization is "faster," which it is, on a dimension nothing in the system was measured to need.

**Example:** PostgreSQL and the Linux kernel both reject speculative micro-optimization of non-critical paths — configuration parsing, rarely-exercised error branches — specifically to preserve reviewability, and require concrete benchmark or production-profiling evidence before a performance-motivated change to a hot path is accepted. The bar is the same one Ch 33 already set for unsafe code: profiling evidence precedes the change, not the other way around.

### Decision: Confirm a System-Level Constraint, or Optimize the Component That Looks Slow

**What it is:** Once the gate in the previous decision is open, where the resulting effort goes: at the specific component that appears slow under local inspection, or at whichever component evidence shows is actually the limiting constraint on the violated requirement.

**Why it exists:** Ch 08 already established that a system's throughput is bounded by its bottleneck, not by the sum or average of its components — and that a local improvement to a non-bottleneck component has no effect on the metric anyone actually experiences. A component "looking slow" under isolated inspection is not evidence that it is the bottleneck; it is evidence that it is slow in isolation, which is a different claim.

**Options:**
- **Local hotspot optimization** — improve whichever component profiles as slow in isolation, or whichever module a developer's intuition points to.
- **Confirmed system-constraint optimization** — use end-to-end measurement (a distributed trace, an SLI breakdown) to identify which component is actually limiting the violated requirement, and direct effort there specifically.

**Trade-offs:** Local hotspot optimization has the lowest discovery cost — no cross-service instrumentation required, just inspect the code that looks slow — but carries a high risk of irrelevance: the fix can land cleanly and change nothing the user or the SLO experiences. Confirmed system-constraint optimization costs more upfront, in tracing infrastructure and cross-team coordination, but every hour spent after that confirmation converges on a measurable improvement, because Ch 08's Theory of Constraints guarantees that capacity gained at the actual bottleneck is capacity gained system-wide.

**When to choose each:** [Consensus] Confirmed system-constraint optimization is the default for any system with more than one interacting component — a service calling a database, a multi-service architecture, anything with a network hop between the code being profiled and the requirement being violated. Local hotspot optimization is defensible only when the component being optimized *is* the entire system as far as the requirement is concerned — a single-process CLI tool, a self-contained library with no external calls on its hot path.

**Common failure modes:** Ch 72's checkout endpoint case is the canonical instance: an engineering team facing a two-second p99 regression on a checkout endpoint spent three weeks profiling locally, trimming memory allocations and object creation, and shaved 15 milliseconds off local CPU time — a rounding error against the regression. A distributed trace on the same endpoint would have shown, in minutes, that the two seconds were one uninterrupted gap between a gateway's outbound call and a downstream service's inbound span: connection-pool exhaustion at the network perimeter, not application code. Three weeks of correct, careful local optimization produced zero improvement to the metric the SLO actually measured, because the component optimized was never the bottleneck.

**Example:** Rewriting a PostgreSQL query or adding an index reliably improves that query's local execution time. If the database is actually constrained by lock contention or disk I/O saturation from a different workload entirely, the query-level win does not move overall throughput — the query was never the system's binding constraint.

### Premature Optimization, Correctly Interpreted

Knuth's line — "premature optimization is the root of all evil" — is almost always quoted as a blanket argument against optimizing at all. The actual passage is narrower: Knuth observed that programmers spend disproportionate effort on the noncritical majority of a program, while a small fraction of the code — he estimated roughly 3% — accounts for the bulk of its runtime cost. The error he identified is not optimization as an activity. It is optimizing before evidence identifies which 3% actually matters, which is exactly the failure the two decisions above are built to prevent.

Read this way, "premature" is not a statement about *when in the project timeline* an optimization happens — it's a statement about *what evidence preceded it*. An optimization made in week one against a documented, already-violated cost budget is not premature. An optimization made in year three against a component nobody has measured is.

### Why Smart Engineers Disagree

The disagreement here is not about whether unjustified optimization wastes effort — everyone agrees it does. It is about how much risk a team can tolerate carrying before a requirement is confirmed to be violated.

Engineers optimizing for time-to-market — typically product and feature teams — push optimization as late as possible, on the grounds that a system's dominant early risk is failing to find product-market fit, not running out of capacity. They accept unoptimized queries, chatty internal calls, and heavyweight framework defaults, because Ch 01 already established that cost of change is the dominant cost for most systems, and speculative performance work spends that budget against a requirement that may never materialize.

Engineers optimizing for platform stability — typically SREs and infrastructure leads — treat waiting for a confirmed SLO breach as waiting for a customer to already be experiencing the failure. They point out that a system can look healthy on averages while its tail latency (Ch 87) quietly rots from a lack of mechanical sympathy in a shared, foundational code path, and that a culture permissive of that rot accumulates a debt that surfaces as a cascading incident, not a gradual one.

Both are reasoning correctly from different risk profiles: the first group is protecting the time-to-market constraint, the second is protecting the safety margin of shared infrastructure. The resolution is not to pick a side globally — it is to apply a tiered version of the same gate: hold shared, foundational infrastructure (the kind speculative tuning was already defensible for, above) to continuous benchmarking against known physical costs, while refusing to touch volatile, user-facing product logic until a real, measured requirement says otherwise.

### The Gate, Stated in Full

Optimization is justified engineering work only when all of the following hold:

1. A real, documented requirement exists — an SLO, a cost envelope, or a measured user-impact threshold — not a feeling that something is slow.
2. That requirement is being violated, or evidence shows it is about to be.
3. A specific system-level constraint has been identified as the cause — not merely a component that looks slow in isolation.

If any of the three is missing, the work in question is speculative refinement, not a response to a real constraint. Every chapter that follows in this Part — how to find the constraint (Ch 86), how to reason about it (Ch 87–89), and what abstraction itself costs once the constraint is found (Ch 90) — assumes this gate has already been passed.
