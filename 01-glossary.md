# Glossary

Authoritative definitions for terms used throughout this handbook. A term gets an entry here when it is used across more than one chapter or when its precise meaning matters to the argument being made.

Terms are added as chapters are completed. If a term is used in a chapter but not yet defined here, that is a gap to fill.

---

**afferent coupling (Ca)**: The number of external components that depend on a given component. High afferent coupling means changes to this component have wide impact. Contrasted with efferent coupling. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**accidental complexity**: Complexity introduced by the engineering solution rather than the problem domain itself. Can be reduced by changing the solution. Contrasted with essential complexity. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**cyclomatic complexity**: A quantitative measure of the number of independent execution paths through a program, derived from the control flow graph. Higher values indicate harder-to-test and harder-to-reason-about code. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**cohesion**: The degree to which the elements inside a single component belong together and serve a single well-defined purpose. High cohesion reduces internal complexity. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**connascence**: A taxonomy for evaluating the strength of coupling between two components. Two components are connascent if a change in one requires a change in the other. Forms range from weak (name, type) to strong (execution order, timing). First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**coupling**: The degree to which one component's behavior depends on another component's state, structure, or timing. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**cost of change**: How expensive it is to modify a system over time, measured in engineering time, risk, and coordination overhead. Distinct from cost of execution. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**distributed monolith**: A system decomposed into multiple services that remain tightly coupled through a shared database schema, undocumented contracts, or implicit timing dependencies. Has the operational complexity of microservices with the coupling of a monolith. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**efferent coupling (Ce)**: The number of external components a given component depends on. High efferent coupling makes a component fragile — vulnerable to changes in any of its dependencies. Contrasted with afferent coupling. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**essential complexity**: Complexity inherent to the problem domain that cannot be eliminated without changing what the system does. Contrasted with accidental complexity. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**instability metric**: I = Ce / (Ca + Ce). A value of 0 indicates a maximally stable component (depended on by many, depends on nothing); a value of 1 indicates a maximally unstable component (depends on many, depended on by nothing). First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**shotgun surgery**: A symptom of low cohesion where a single conceptual change requires modifying many files scattered across a codebase. Indicates the concept is not owned by any one component. First introduced in: [Part I, Ch 03](part1-systems-thinking/ch03-coupling-and-cohesion.md).

**state space explosion**: The condition where mutable state variables combine to produce a number of possible system configurations that exceeds what engineers can anticipate or test. A primary failure mode of unconstrained mutable state. First introduced in: [Part I, Ch 02](part1-systems-thinking/ch02-complexity-is-the-enemy.md).

**MTBF (Mean Time Between Failures)**: A reliability paradigm that optimizes for preventing failures from occurring. Contrasted with MTTR. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**MTTR (Mean Time To Recovery)**: A reliability paradigm that accepts failures as inevitable and optimizes for recovering from them quickly. Contrasted with MTBF. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**optimization target**: An objective a system is designed to optimize — latency, throughput, reliability, cost of change, etc. Targets may be explicit (documented) or implicit (inferred). First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

**optimization target drift**: The phenomenon where a system's actual optimization targets diverge from its intended ones over time due to accumulated changes, hotfixes, and operational adjustments. First introduced in: [Part I, Ch 01](part1-systems-thinking/ch01-what-engineering-optimizes.md).

---

## Format

Each entry follows:

```
**Term**: Definition in one to three sentences.
First introduced in: [Part X, Chapter Y](link).
```
