# Glossary

Authoritative definitions for terms used throughout this handbook. A term gets an entry here when it is used across more than one chapter or when its precise meaning matters to the argument being made.

Terms are added as chapters are completed. If a term is used in a chapter but not yet defined here, that is a gap to fill.

---

**afferent coupling (Ca)**: The number of external components that depend on a given component. High afferent coupling means changes to this component have wide impact. Contrasted with efferent coupling. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**axis of variation**: A specific, named dimension along which a system is expected to change — an authentication method, a storage backend, a data format. Identifying real axes of variation is what distinguishes designing for change from future-proofing. First introduced in: [Part I, Ch 05](part1-systems-thinking/ch05-designing-for-change.md).

**accidental complexity**: Complexity introduced by the engineering solution rather than the problem domain itself. Can be reduced by changing the solution. Contrasted with essential complexity. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**cyclomatic complexity**: A quantitative measure of the number of independent execution paths through a program, derived from the control flow graph. Higher values indicate harder-to-test and harder-to-reason-about code. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**cohesion**: The degree to which the elements inside a single component belong together and serve a single well-defined purpose. High cohesion reduces internal complexity. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**connascence**: A taxonomy for evaluating the strength of coupling between two components. Two components are connascent if a change in one requires a change in the other. Forms range from weak (name, type) to strong (execution order, timing). First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**coupling**: The degree to which one component's behavior depends on another component's state, structure, or timing. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**cost of change**: How expensive it is to modify a system over time, measured in engineering time, risk, and coordination overhead. Distinct from cost of execution. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**distributed monolith**: A system decomposed into multiple services that remain tightly coupled through a shared database schema, undocumented contracts, or implicit timing dependencies. Has the operational complexity of microservices with the coupling of a monolith. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**encapsulation**: The mechanical bundling of data with the methods that operate on it. A language feature, not an architectural guarantee — encapsulation hides implementation but does not by itself hide volatile design decisions. Contrasted with information hiding. First introduced in: [Part I, Ch 04](part1-systems-thinking/ch04-abstraction-and-information-hiding.md).

**efferent coupling (Ce)**: The number of external components a given component depends on. High efferent coupling makes a component fragile — vulnerable to changes in any of its dependencies. Contrasted with afferent coupling. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**essential complexity**: Complexity inherent to the problem domain that cannot be eliminated without changing what the system does. Contrasted with accidental complexity. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**future-proofing**: Speculative anticipation of unknown future requirements through upfront generality, as distinct from designing for change along a known, specific axis of variation. Usually an anti-pattern — the complexity cost is paid immediately for a benefit that may never arrive. First introduced in: [Part I, Ch 05](part1-systems-thinking/ch05-designing-for-change.md).

**latency hierarchy**: The approximate cost, in time, of retrieving data from each layer of a real system — CPU cache, RAM, SSD, and network — spanning many orders of magnitude rather than a smooth gradient. Memorized by systems engineers as a baseline for reasoning about architectural cost. First introduced in: [Part I, Ch 06](part1-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md).

**mechanical sympathy**: The principle, popularized by Martin Thompson, that software performs better when it respects how the underlying hardware actually executes instructions, moves data, and manages memory, rather than fighting those properties for the sake of abstract code cleanliness. First introduced in: [Part I, Ch 06](part1-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md).

**MVCC (Multi-Version Concurrency Control)**: A concurrency strategy where every update creates a new version of a row rather than mutating it in place, allowing readers and writers to proceed without blocking each other at the cost of storage overhead and background cleanup (vacuuming). Contrasted with pessimistic locking. First introduced in: [Part I, Ch 06](part1-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md).

**information hiding**: The deliberate concealment of a design decision that is likely to change, as distinct from merely hiding implementation. Coined by Parnas (1972). The goal is to isolate volatility behind a stable interface so that an internal change does not propagate to callers. Contrasted with encapsulation. First introduced in: [Part I, Ch 04](part1-systems-thinking/ch04-abstraction-and-information-hiding.md).

**instability metric**: I = Ce / (Ca + Ce). A value of 0 indicates a maximally stable component (depended on by many, depends on nothing); a value of 1 indicates a maximally unstable component (depends on many, depended on by nothing). First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**leaky abstraction**: Spolsky's Law (2002): all non-trivial abstractions, to some degree, leak — the implementation details they were meant to hide eventually surface under load, failure, or scale. The design question is not whether an abstraction will leak, but when, how badly, and whether the system was built to survive it. First introduced in: [Part I, Ch 04](part1-systems-thinking/ch04-abstraction-and-information-hiding.md).

**Rule of Three**: A heuristic for deferring abstraction: tolerate duplication until a third genuinely similar use case appears before extracting a shared interface. Guards against premature abstraction built on too little evidence of an actual pattern. First introduced in: [Part I, Ch 04](part1-systems-thinking/ch04-abstraction-and-information-hiding.md).

**shotgun surgery**: A symptom of low cohesion where a single conceptual change requires modifying many files scattered across a codebase. Indicates the concept is not owned by any one component. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**state space explosion**: The condition where mutable state variables combine to produce a number of possible system configurations that exceeds what engineers can anticipate or test. A primary failure mode of unconstrained mutable state. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**MTBF (Mean Time Between Failures)**: A reliability paradigm that optimizes for preventing failures from occurring. Contrasted with MTTR. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**MTTR (Mean Time To Recovery)**: A reliability paradigm that accepts failures as inevitable and optimizes for recovering from them quickly. Contrasted with MTBF. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**optimization target**: An objective a system is designed to optimize — latency, throughput, reliability, cost of change, etc. Targets may be explicit (documented) or implicit (inferred). First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**optimization target drift**: The phenomenon where a system's actual optimization targets diverge from its intended ones over time due to accumulated changes, hotfixes, and operational adjustments. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**Open/Closed Principle (OCP)**: Software entities should be open for extension but closed for modification — new behavior is added through new code rather than by editing existing, tested code. A pragmatic lens for managing regression risk where change is genuinely additive, not a mandate to apply everywhere. First introduced in: [Part I, Ch 05](part1-systems-thinking/ch05-designing-for-change.md).

**wrong abstraction**: An abstraction built on an incorrect guess about what will change, which couples every caller to a false model of the problem. Worse than no abstraction at all, because unwinding the coupling costs more than the duplication it was meant to prevent. First introduced in: [Part I, Ch 04](part1-systems-thinking/ch04-abstraction-and-information-hiding.md).

---

## Format

Each entry follows:

```
**Term**: Definition in one to three sentences.
First introduced in: [Part X, Chapter Y](link).
```
