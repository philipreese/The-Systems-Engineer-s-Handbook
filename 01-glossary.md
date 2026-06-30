# Glossary

Authoritative definitions for terms used throughout this handbook. A term gets an entry here when it is used across more than one chapter or when its precise meaning matters to the argument being made.

Terms are added as chapters are completed. If a term is used in a chapter but not yet defined here, that is a gap to fill.

---

**Architecture Decision Record (ADR)**: A lightweight document stored in the repository that records what was decided, the context and constraints present at decision time, and what alternatives were rejected and why. The mechanism by which a decision's reasoning outlives the people who made it. First introduced in: [Part I, Ch 09](part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md).

**afferent coupling (Ca)**: The number of external components that depend on a given component. High afferent coupling means changes to this component have wide impact. Contrasted with efferent coupling. First introduced in: [Part I, Ch 03](part01-systems-thinking/ch03-coupling-and-cohesion.md).

**axis of variation**: A specific, named dimension along which a system is expected to change — an authentication method, a storage backend, a data format. Identifying real axes of variation is what distinguishes designing for change from future-proofing. First introduced in: [Part I, Ch 05](part01-systems-thinking/ch05-designing-for-change.md).

**backpressure**: A flow-control mechanism that enforces a hard upper bound on queue depth and rejects new requests once that bound is reached, preventing queue exhaustion and node crashes at the cost of requiring upstream callers to handle rejection. Contrasted with unbounded queues. First introduced in: [Part I, Ch 08](part01-systems-thinking/ch08-local-vs-global-optimization.md). A specific, named dimension along which a system is expected to change — an authentication method, a storage backend, a data format. Identifying real axes of variation is what distinguishes designing for change from future-proofing. First introduced in: [Part I, Ch 05](part01-systems-thinking/ch05-designing-for-change.md).

**blast radius**: The scope of damage a wrong decision causes to the broader system or business. Combined with reversibility, determines how much deliberation a decision warrants. First introduced in: [Part I, Ch 09](part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md).

**accidental complexity**: Complexity introduced by the engineering solution rather than the problem domain itself. Can be reduced by changing the solution. Contrasted with essential complexity. First introduced in: [Part I, Ch 02](part01-systems-thinking/ch02-complexity-is-the-enemy.md).

**the action problem**: The recurring case in resource-oriented API design where a real business operation — canceling an order, refunding a payment — doesn't map cleanly onto standard CRUD verbs (GET/PUT/PATCH/DELETE), forcing a choice between generic state patching, an explicit sub-resource action endpoint, or a separate action resource. First introduced in: [Part III, Ch 20](part03-api-design/ch20-resource-modeling.md).

**Cynefin framework**: A sense-making model (Snowden) that classifies problems by the relationship between cause and effect: Simple (best practice applies), Complicated (analysis required, right answer exists), Complex (probe-sense-respond, no single right answer), Chaotic (act first to restore order). Most architectural decisions are complicated. First introduced in: [Part I, Ch 09](part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md).

**cursor (keyset pagination)**: A stable, unique, sortable pointer to a specific position in a collection, used to anchor a pagination request ("items after this one") instead of a relative offset counted from the start. Lets the database jump directly to the position via an index, giving constant performance regardless of depth and immunity to page drift. First introduced in: [Part III, Ch 23](part03-api-design/ch23-pagination-and-streaming.md).

**cyclomatic complexity**: A quantitative measure of the number of independent execution paths through a program, derived from the control flow graph. Higher values indicate harder-to-test and harder-to-reason-about code. First introduced in: [Part I, Ch 02](part01-systems-thinking/ch02-complexity-is-the-enemy.md).

**cohesion**: The degree to which the elements inside a single component belong together and serve a single well-defined purpose. High cohesion reduces internal complexity. First introduced in: [Part I, Ch 03](part01-systems-thinking/ch03-coupling-and-cohesion.md).

**confused deputy problem**: A failure mode where a service uses its own elevated, perimeter-trusted privileges to act on a caller's behalf without verifying that the caller was actually authorized to request that action — the service isn't compromised, it's tricked into misusing its own authority. First introduced in: [Part III, Ch 24](part03-api-design/ch24-authentication-authorization-boundaries.md).

**connascence**: A taxonomy for evaluating the strength of coupling between two components. Two components are connascent if a change in one requires a change in the other. Forms range from weak (name, type) to strong (execution order, timing). First introduced in: [Part I, Ch 03](part01-systems-thinking/ch03-coupling-and-cohesion.md).

**coupling**: The degree to which one component's behavior depends on another component's state, structure, or timing. First introduced in: [Part I, Ch 03](part01-systems-thinking/ch03-coupling-and-cohesion.md).

**CAP theorem**: In the presence of a network partition, a distributed system must choose between consistency (every read receives the most recent write or an error) and availability (every request receives a non-error response). Partition tolerance is not optional in real distributed systems. First introduced in: [Part I, Ch 07](part01-systems-thinking/ch07-reliability-as-a-design-principle.md).

**Conway's Law**: Organizations that design systems are constrained to produce designs that mirror their own communication structures. Local team autonomy produces systems with those team boundaries baked in — which may not be the correct system boundaries. First introduced in: [Part I, Ch 08](part01-systems-thinking/ch08-local-vs-global-optimization.md).

**consumer-driven contract test**: A test, authored from the consumer's expectations of an API rather than the provider's implementation, that runs against the provider to verify a proposed change won't break that specific known consumer before it ships. The mechanism formal change management uses to replace direct, synchronous coordination once consumers can't be reached directly. First introduced in: [Part III, Ch 25](part03-api-design/ch25-internal-vs-external-api-design.md).

**correlation ID**: A unique identifier included in an error response (and propagated through internal logs and traces) that lets a caller hand a single failure back to the provider and have it matched to the exact request that produced it. First introduced in: [Part III, Ch 21](part03-api-design/ch21-error-handling-contracts.md).

**cost of change**: How expensive it is to modify a system over time, measured in engineering time, risk, and coordination overhead. Distinct from cost of execution. First introduced in: [Part I, Ch 01](part01-systems-thinking/ch01-what-engineering-optimizes.md).

**distributed monolith**: A system decomposed into multiple services that remain tightly coupled through a shared database schema, undocumented contracts, or implicit timing dependencies. Has the operational complexity of microservices with the coupling of a monolith. First introduced in: [Part I, Ch 03](part01-systems-thinking/ch03-coupling-and-cohesion.md).

**encapsulation**: The mechanical bundling of data with the methods that operate on it. A language feature, not an architectural guarantee — encapsulation hides implementation but does not by itself hide volatile design decisions. Contrasted with information hiding. First introduced in: [Part I, Ch 04](part01-systems-thinking/ch04-abstraction-and-information-hiding.md).

**efferent coupling (Ce)**: The number of external components a given component depends on. High efferent coupling makes a component fragile — vulnerable to changes in any of its dependencies. Contrasted with afferent coupling. First introduced in: [Part I, Ch 03](part01-systems-thinking/ch03-coupling-and-cohesion.md).

**essential complexity**: Complexity inherent to the problem domain that cannot be eliminated without changing what the system does. Contrasted with accidental complexity. First introduced in: [Part I, Ch 02](part01-systems-thinking/ch02-complexity-is-the-enemy.md).

**fail-fast**: The design principle of terminating execution immediately upon detecting an invalid or inconsistent state, rather than continuing in a potentially corrupted state. Converts wrong-answer failures into crash failures — visible and bounded rather than silent and spreading. First introduced in: [Part I, Ch 07](part01-systems-thinking/ch07-reliability-as-a-design-principle.md).

**future-proofing**: Speculative anticipation of unknown future requirements through upfront generality, as distinct from designing for change along a known, specific axis of variation. Usually an anti-pattern — the complexity cost is paid immediately for a benefit that may never arrive. First introduced in: [Part I, Ch 05](part01-systems-thinking/ch05-designing-for-change.md).

**latency hierarchy**: The approximate cost, in time, of retrieving data from each layer of a real system — CPU cache, RAM, SSD, and network — spanning many orders of magnitude rather than a smooth gradient. Memorized by systems engineers as a baseline for reasoning about architectural cost. First introduced in: [Part I, Ch 06](part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md).

**Little's Law**: L = λW. In any stable queuing system, queue depth (L) equals arrival rate (λ) multiplied by average time in system (W). Explains why reducing local latency can increase arrival rate at a downstream component, growing queue depth and worsening system-level latency. First introduced in: [Part I, Ch 08](part01-systems-thinking/ch08-local-vs-global-optimization.md). The approximate cost, in time, of retrieving data from each layer of a real system — CPU cache, RAM, SSD, and network — spanning many orders of magnitude rather than a smooth gradient. Memorized by systems engineers as a baseline for reasoning about architectural cost. First introduced in: [Part I, Ch 06](part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md).

**mechanical sympathy**: The principle, popularized by Martin Thompson, that software performs better when it respects how the underlying hardware actually executes instructions, moves data, and manages memory, rather than fighting those properties for the sake of abstract code cleanliness. First introduced in: [Part I, Ch 06](part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md).

**MVCC (Multi-Version Concurrency Control)**: A concurrency strategy where every update creates a new version of a row rather than mutating it in place, allowing readers and writers to proceed without blocking each other at the cost of storage overhead and background cleanup (vacuuming). Contrasted with pessimistic locking. First introduced in: [Part I, Ch 06](part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md).

**idempotency key**: A unique identifier, generated by the client per logical operation, that the server uses to recognize a retried request and return the original cached result instead of re-executing its side effect. Enforced correctly with a database uniqueness constraint, written atomically with the side effect, never with a separate read-then-write check. First introduced in: [Part III, Ch 22](part03-api-design/ch22-idempotency.md).

**information hiding**: The deliberate concealment of a design decision that is likely to change, as distinct from merely hiding implementation. Coined by Parnas (1972). The goal is to isolate volatility behind a stable interface so that an internal change does not propagate to callers. Contrasted with encapsulation. First introduced in: [Part I, Ch 04](part01-systems-thinking/ch04-abstraction-and-information-hiding.md).

**instability metric**: I = Ce / (Ca + Ce). A value of 0 indicates a maximally stable component (depended on by many, depends on nothing); a value of 1 indicates a maximally unstable component (depends on many, depended on by nothing). First introduced in: [Part I, Ch 03](part01-systems-thinking/ch03-coupling-and-cohesion.md).

**leaky abstraction**: Spolsky's Law (2002): all non-trivial abstractions, to some degree, leak — the implementation details they were meant to hide eventually surface under load, failure, or scale. The design question is not whether an abstraction will leak, but when, how badly, and whether the system was built to survive it. First introduced in: [Part I, Ch 04](part01-systems-thinking/ch04-abstraction-and-information-hiding.md).

**Rule of Three**: A heuristic for deferring abstraction: tolerate duplication until a third genuinely similar use case appears before extracting a shared interface. Guards against premature abstraction built on too little evidence of an actual pattern. First introduced in: [Part I, Ch 04](part01-systems-thinking/ch04-abstraction-and-information-hiding.md).

**shotgun surgery**: A symptom of low cohesion where a single conceptual change requires modifying many files scattered across a codebase. Indicates the concept is not owned by any one component. First introduced in: [Part I, Ch 03](part01-systems-thinking/ch03-coupling-and-cohesion.md).

**state space explosion**: The condition where mutable state variables combine to produce a number of possible system configurations that exceeds what engineers can anticipate or test. A primary failure mode of unconstrained mutable state. First introduced in: [Part I, Ch 02](part01-systems-thinking/ch02-complexity-is-the-enemy.md).

**MTBF (Mean Time Between Failures)**: A reliability paradigm that optimizes for preventing failures from occurring. Contrasted with MTTR. First introduced in: [Part I, Ch 01](part01-systems-thinking/ch01-what-engineering-optimizes.md).

**MTTR (Mean Time To Recovery)**: A reliability paradigm that accepts failures as inevitable and optimizes for recovering from them quickly. Contrasted with MTBF. First introduced in: [Part I, Ch 01](part01-systems-thinking/ch01-what-engineering-optimizes.md).

**optimization target**: An objective a system is designed to optimize — latency, throughput, reliability, cost of change, etc. Targets may be explicit (documented) or implicit (inferred). First introduced in: [Part I, Ch 01](part01-systems-thinking/ch01-what-engineering-optimizes.md).

**partial failure**: The condition in distributed systems where some components succeed and others fail simultaneously, producing a state that is neither success nor failure globally. The normal operational mode of distributed systems, not an edge case. First introduced in: [Part I, Ch 07](part01-systems-thinking/ch07-reliability-as-a-design-principle.md). An objective a system is designed to optimize — latency, throughput, reliability, cost of change, etc. Targets may be explicit (documented) or implicit (inferred). First introduced in: [Part I, Ch 01](part01-systems-thinking/ch01-what-engineering-optimizes.md).

**page drift**: The instability that offset-based pagination suffers under concurrent writes — an insert or delete shifts which items fall on which page between requests, causing a client to silently see duplicates or skip items entirely. First introduced in: [Part III, Ch 23](part03-api-design/ch23-pagination-and-streaming.md).

**optimization target drift**: The phenomenon where a system's actual optimization targets diverge from its intended ones over time due to accumulated changes, hotfixes, and operational adjustments. First introduced in: [Part I, Ch 01](part01-systems-thinking/ch01-what-engineering-optimizes.md).

**Open/Closed Principle (OCP)**: Software entities should be open for extension but closed for modification — new behavior is added through new code rather than by editing existing, tested code. A pragmatic lens for managing regression risk where change is genuinely additive, not a mandate to apply everywhere. First introduced in: [Part I, Ch 05](part01-systems-thinking/ch05-designing-for-change.md).

**reversibility**: How expensive it is to undo a decision. Combined with blast radius, determines how much deliberation a decision warrants. Low-reversibility / high-blast-radius decisions require heavy deliberation; high-reversibility / low-blast-radius decisions should be made quickly. First introduced in: [Part I, Ch 09](part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md).

**Theory of Constraints**: Goldratt's principle that system throughput is bounded by the single slowest component (the bottleneck). Optimizing any non-bottleneck component has no effect on system throughput and often increases load on the actual bottleneck. First introduced in: [Part I, Ch 08](part01-systems-thinking/ch08-local-vs-global-optimization.md).

**Write-Ahead Log (WAL)**: A durability mechanism where a database appends transaction intent to a sequential log before updating data files, enabling durability guarantees without paying the latency cost of random I/O on every commit. On crash, the log is replayed to recover committed transactions. First introduced in: [Part I, Ch 07](part01-systems-thinking/ch07-reliability-as-a-design-principle.md).

**zero-trust architecture**: A security model that re-verifies identity and authorization at every service boundary instead of trusting requests based on network location alone. The structural response to the confused deputy problem — no internal call is trusted just because it originated inside the perimeter. First introduced in: [Part III, Ch 24](part03-api-design/ch24-authentication-authorization-boundaries.md).

**wrong abstraction**: An abstraction built on an incorrect guess about what will change, which couples every caller to a false model of the problem. Worse than no abstraction at all, because unwinding the coupling costs more than the duplication it was meant to prevent. First introduced in: [Part I, Ch 04](part01-systems-thinking/ch04-abstraction-and-information-hiding.md).

**anti-corruption layer (ACL)**: A translation boundary, from Domain-Driven Design, placed between an internal bounded context and an external or legacy system whose model and vocabulary are incompatible with it. It translates the external system's shape into the internal domain's own vocabulary so foreign constraints never enter the core. First introduced in: [Part II, Ch 14](part02-software-architecture/ch14-abstraction-layers-when-to-add-one.md).

**big ball of mud**: A monolith with no enforced internal module boundaries, where any component can call any other. The degenerate case a modular monolith is meant to prevent. First introduced in: [Part II, Ch 10](part02-software-architecture/ch10-monolith-vs-service-decomposition.md).

**bounded context**: The largest architectural scope, from Domain-Driven Design (Eric Evans), within which a domain model's terms and rules remain internally consistent. The unit a service boundary should be drawn around. First introduced in: [Part II, Ch 13](part02-software-architecture/ch13-coupling-cohesion-architecture-level.md).

**compensating action**: An explicit action that reverses the effect of a previously completed step in a distributed workflow, used in place of a database rollback when a multi-step process spans more than one service. First introduced in: [Part II, Ch 17](part02-software-architecture/ch17-sync-vs-async-communication.md).

**CQRS (Command Query Responsibility Segregation)**: An architectural pattern that separates the model used to write data from the model used to read it, commonly used across service boundaries to build a fast, local, read-optimized projection of data another service owns. First introduced in: [Part II, Ch 18](part02-software-architecture/ch18-data-ownership-boundaries.md).

**database-per-service pattern**: The rule that each service's datastore is reachable only by that service; every other service accesses its data exclusively through its API. The hard prerequisite for treating a decomposed service as actually independent. First introduced in: [Part II, Ch 18](part02-software-architecture/ch18-data-ownership-boundaries.md).

**Data Transfer Object (DTO)**: A data structure designed specifically for a boundary crossing — an API request or response — distinct from the domain model or database entity it is mapped to or from. First introduced in: [Part II, Ch 11](part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md).

**Dependency Inversion Principle (DIP)**: High-level (stable) modules must not depend on low-level (volatile) modules; both should depend on an abstraction, and that abstraction must be owned by the high-level side. First introduced in: [Part II, Ch 12](part02-software-architecture/ch12-dependency-direction-inversion.md).

**event-carried state transfer**: An event payload design where the message carries the full state of the resource at the moment of the event ("a fat event"), rather than only an identifier — letting consumers act without calling back to the publisher, at the cost of risking eventual-consistency collapse if delivery lags behind newer mutations. First introduced in: [Part III, Ch 19](part03-api-design/ch19-rest-vs-rpc-vs-event-driven.md).

**HATEOAS (Hypermedia As The Engine Of Application State)**: REST's textbook ideal in which server responses embed the links describing every action currently available, so a client never needs prior knowledge of the API's URL structure. Almost never implemented in production — the coordination and tooling cost rarely outweighs disciplined, documented, pragmatic REST. First introduced in: [Part III, Ch 19](part03-api-design/ch19-rest-vs-rpc-vs-event-driven.md).

**Hyrum's Law**: With a sufficient number of consumers, every observable behavior of a system — whether documented as part of the contract or not — will eventually be depended on by somebody. Coined at Google. The mechanism behind accidental externalization: an undocumented sort order or incidental field becomes an implicit contract the moment enough consumers rely on it. First introduced in: [Part III, Ch 25](part03-api-design/ch25-internal-vs-external-api-design.md).

**hexagonal architecture (ports-and-adapters)**: An internal architecture, introduced by Alistair Cockburn, in which business logic sits at the center and defines the interfaces ("ports") it needs; infrastructure connects to the core by implementing those interfaces ("adapters"), so dependencies always point inward. First introduced in: [Part II, Ch 11](part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md).

**layer leakage**: The failure mode where infrastructure or implementation details surface above the layer meant to hide them — for example, a SQL exception reaching the API layer — defeating the information hiding the layering was supposed to provide. First introduced in: [Part II, Ch 11](part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md).

**modular monolith**: A single deployable unit with strictly enforced internal module boundaries and information hiding, as distinct from a big ball of mud. The recommended default starting architecture for most systems. First introduced in: [Part II, Ch 10](part02-software-architecture/ch10-monolith-vs-service-decomposition.md).

**pass-through layer**: An architectural layer that forwards a call unchanged, hiding no decision and performing no transformation — the dominant failure mode of adding layers as a default rather than in response to real volatility. First introduced in: [Part II, Ch 14](part02-software-architecture/ch14-abstraction-layers-when-to-add-one.md).

**progressive disclosure**: An API design strategy where the default request and response are as simple as possible, with advanced capability available through explicit optional parameters or expansion rather than present by default. First introduced in: [Part II, Ch 15](part02-software-architecture/ch15-api-surface-design-expose-hide.md).

**repository pattern**: An interface, owned by the domain, that abstracts data access behind domain vocabulary (e.g., `findUserById`) rather than a specific database client, allowing the underlying storage implementation to be swapped without changing the code that depends on it. First introduced in: [Part II, Ch 12](part02-software-architecture/ch12-dependency-direction-inversion.md).

**saga pattern**: A pattern for maintaining consistency across a multi-step workflow spanning several services, using a sequence of local transactions and explicit compensating actions instead of a single atomic database transaction. First introduced in: [Part II, Ch 17](part02-software-architecture/ch17-sync-vs-async-communication.md).

**strangler fig pattern**: An incremental migration pattern, named by Martin Fowler, in which a new service is grown around the edge of an existing monolith — intercepting a growing slice of live traffic — until the corresponding functionality can be removed from the monolith entirely. First introduced in: [Part II, Ch 10](part02-software-architecture/ch10-monolith-vs-service-decomposition.md).

**sunset pattern**: A structured, time-bound lifecycle for retiring an obsolete API version — active, deprecated, sunset, removal — communicated through escalating, programmatic signals rather than documentation alone. First introduced in: [Part II, Ch 16](part02-software-architecture/ch16-versioning-backward-compatibility.md).

**temporal coupling**: The requirement that a caller and a callee both be available, healthy, and reachable at the same instant for an interaction to succeed — the defining property of synchronous communication, and the architectural expression of connascence of execution order. First introduced in: [Part II, Ch 13](part02-software-architecture/ch13-coupling-cohesion-architecture-level.md).

---

## Format

Each entry follows:

```
**Term**: Definition in one to three sentences.
First introduced in: [Part X, Chapter Y](link).
```
