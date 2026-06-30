# Part II — Chapter Specifications

Pre-filled special instructions for each Part II chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part II: Part I. Readers are expected to know the full Part I vocabulary:
complexity, coupling, cohesion, abstraction, information hiding, axes of variation, the
latency hierarchy, fail-fast, partial failure, Little's Law, Theory of Constraints.

---

## Ch 10 — Monolith vs. Service Decomposition

```
This chapter establishes when service decomposition is warranted and when it is not. The
argument to make explicitly: decomposition is not inherently good. Most systems should start
as a monolith. Extraction into services is justified when a specific, named constraint
demands it — not because "microservices are modern."

Cover the real costs of decomposition: distributed failure modes (partial failure from Ch 07),
network latency tax (from Ch 06), operational overhead, and the distributed monolith
anti-pattern (Ch 03 vocabulary) as the primary failure mode. Cover the real benefits: 
independent scaling, independent deployment, and team autonomy (Conway's Law from Ch 08).

Name the concrete signals that justify decomposition: a component that needs to scale
independently at a different rate than the rest of the system; a component with a genuinely
different failure domain or reliability requirement; a team boundary that maps cleanly to a
domain boundary.

Anchor in real systems: Shopify's modular monolith (deliberate choice to stay unified at
scale), Stack Overflow (single server handles massive load — the monolith as a deliberate
optimization), Amazon/Netflix as the canonical microservices case study, and the "strangler
fig pattern" (Martin Fowler) as the safe path from monolith to services.

Do NOT cover: specific internal architecture patterns like hexagonal or ports-and-adapters
(Ch 11), the mechanics of dependency inversion (Ch 12), API versioning strategy (Ch 16),
data ownership patterns in detail (Ch 18 covers that), synchronous vs. async communication
mechanics (Ch 17). This chapter is about the decomposition decision itself — not what the
architecture looks like once you've decided.
```

---

## Ch 11 — Layered, Hexagonal, and Ports-and-Adapters Architecture

```
This chapter covers the three primary structural patterns for organizing code within a
service or application boundary. The goal is not to catalog patterns but to explain what
problem each pattern is solving and when each is the right tool.

Layered (n-tier) architecture: the classic presentation/business-logic/data separation.
Explain why it became dominant (simplicity, familiarity), and its failure mode: leakage
across layers that turns the architecture into a dependency tangle over time.

Hexagonal architecture (Alistair Cockburn, 2005) / ports-and-adapters: the core insight is
that business logic should have no dependency on infrastructure. "Ports" are the interfaces
the core defines; "adapters" are the implementations that connect to external systems.
This is information hiding (Ch 04) applied architecturally — the core is insulated from
the volatility of infrastructure choices.

Cover the practical implication: business logic becomes independently testable without
infrastructure. Cover the cost: more upfront interface definition, more indirection.

Anchor in real systems: Django's MTV pattern as a layered architecture, a REST API service
restructured into hexagonal form to illustrate the difference, Spring's layered vs. clean
architecture configurations.

Do NOT cover: the decomposition decision itself (Ch 10), dependency inversion as a formal
principle (Ch 12 goes deeper there), module and file structure within a layer (Part IV
Ch 27), API surface design at the boundary (Ch 15). This chapter is about internal
structural patterns within a service, not cross-service architecture.
```

---

## Ch 12 — Dependency Direction and Inversion

```
This chapter makes the Dependency Inversion Principle (DIP) a concrete, actionable tool
rather than an OOP abstraction. The core argument: in any architecture, the direction of
dependencies determines the direction of change propagation. Stable things must not depend
on volatile things. Dependency inversion reverses that direction by inserting an interface
owned by the stable component.

Cover the dependency rule from Robert Martin's "Clean Architecture": source code dependencies
must always point inward — toward higher-level policy. Concrete implementations at the
edges; abstract interfaces at the core.

Cover how DIP applies at the architecture level, not just the class level: a business logic
module that imports a database driver directly is coupled to a volatile implementation; one
that depends on a `Repository` interface it defines itself controls that coupling direction.

The practical test: can you swap a dependency (database, HTTP client, file system) without
changing the core? If yes, inversion is working. If no, the dependency is pointing the
wrong way.

Anchor in real systems: the repository pattern in Go and Python (concrete implementations
vs. interface-defined dependencies), the "infrastructure at the edges" principle in practice,
PostgreSQL being swappable behind a repository interface (even if it never actually gets
swapped — the design is the point).

Do NOT cover: specific architecture patterns like hexagonal/ports-and-adapters (Ch 11
covers those — DIP is the underlying principle they apply), OCP in detail (Ch 05), module
file structure (Part IV Ch 27), API versioning (Ch 16). This chapter explains the
dependency direction principle; Ch 11 shows it embedded in specific patterns.
```

---

## Ch 13 — Coupling and Cohesion at the Architecture Level

```
Part I Ch 03 established coupling and cohesion as concepts. This chapter applies them
specifically at the architectural / service-boundary level, where the stakes are higher
because interfaces are harder to change and failures are distributed.

The primary question: where do service boundaries belong? The answer from domain-driven
design: around bounded contexts — the largest scope within which a domain model is
internally consistent. A bounded context is the architectural expression of high cohesion:
everything inside shares a common language and model; the interface to the outside world
is narrow and intentional.

Cover the shared database anti-pattern as the canonical high-coupling failure at the
architecture level: two services sharing a PostgreSQL schema are tightly coupled through
the schema regardless of their independent deployment. Any schema change requires
coordinating both.

Cover event-driven decoupling as the primary mechanism for reducing temporal coupling
between services — but be honest about what it adds: eventual consistency, harder
debugging, and the partial-failure failure modes from Ch 07.

Anchor in real systems: DDD bounded contexts (Eric Evans), the shared database anti-pattern
in microservices, Kafka as a decoupling mechanism between services.

Do NOT cover: the initial decomposition decision (Ch 10), specific messaging mechanics
like exactly-once delivery or ordering guarantees (Ch 17), data ownership patterns (Ch 18),
coupling at the code level (Ch 03 already covers that). This chapter applies coupling
concepts to service boundaries specifically.
```

---

## Ch 14 — Abstraction Layers: When to Add One

```
Part I Ch 04 established what abstraction is and why wrong abstractions are worse than
none. This chapter applies that principle at the architectural level: when does adding a
new layer to the architecture reduce complexity, and when does it add indirection without
benefit?

The core test: a new architectural layer is justified when it hides a decision that is
genuinely likely to change and that, without the layer, would require coordinating changes
across many callers. It is not justified by "this seems like it could be useful" or
"other architectures have this layer."

Cover the anti-corruption layer (DDD) as a case where a layer is clearly justified:
integrating a legacy system or third-party service with an inconsistent model. The layer
translates between models so internal code never knows what the external shape is.

Cover the cost of every additional layer: indirection (following the call requires crossing
one more boundary), additional testing surface, and the risk of the layer becoming a
pass-through that adds no value.

The "pass-through layer" is the primary failure mode: a service layer that does nothing
but call the repository method of the same name. It adds no logic, hides no decision, and
exists only because someone thought layers were good by default.

Anchor in real systems: the OSI model as a case where abstraction layers proved their
worth (each layer genuinely hides something the layers above it shouldn't need to know),
the DDD anti-corruption layer pattern, a service-layer-as-pass-through as the failure case.

Do NOT cover: the abstraction concept itself (Ch 04), specific architecture patterns
(Ch 11), API surface design at a boundary (Ch 15), module file structure (Part IV Ch 27).
This chapter answers "when to add a layer" — the surrounding chapters answer what patterns
to use and how to design the resulting interface.
```

---

## Ch 15 — API Surface Design: What to Expose, What to Hide

```
This chapter applies information hiding (Part I Ch 04) to API boundaries specifically.
Every field, parameter, and operation you expose in an API is a commitment — a promise
you must honor in perpetuity or break with coordination cost. The practical implication:
design API surfaces to be as small as possible while still being useful.

Cover the principle of minimal surface area: don't expose fields because they exist
internally; expose them because the caller genuinely needs them and you are willing to
maintain them. Every addition is cheap; every removal or modification is expensive once
consumers exist.

Cover progressive disclosure: APIs can be designed in layers, where the common path is
clean and simple, and advanced capabilities are available but not surfaced prominently.
This is opposed to the "god resource" where every field is always returned.

Cover the distinction between internal and external APIs. Internal APIs (between services
in the same organization) can evolve more aggressively because consumers are known and
coordinated. External APIs (published to third parties) require much stronger stability
guarantees because consumer coordination is impossible.

Anchor in real systems: Stripe's API design (minimal, stable, consistent — and how they
deprecate fields slowly over years), POSIX as an API that has been stable for 50 years
by being radically minimal, GraphQL as a case study in the tradeoffs of consumer-driven
field selection.

Do NOT cover: versioning strategies for evolving an existing API (Ch 16), the REST vs.
RPC vs. event-driven choice (Part III Ch 19), authentication and authorization at the
boundary (Part III Ch 24), module file structure of the API implementation (Part IV Ch 27).
This chapter is about what to put in the API surface, not how to version or transport it.
```

---

## Ch 16 — Versioning and Backward Compatibility

```
Once an API is consumed by external systems, it carries a backward compatibility obligation.
This chapter covers the mechanics of meeting that obligation: how to evolve an API without
breaking consumers, and how to communicate that a contract is changing.

Cover the fundamental distinction: backward compatible changes (adding optional fields,
adding new endpoints, relaxing constraints) vs. breaking changes (removing fields, changing
types, tightening constraints). Most versioning problems are caused by engineers not knowing
this distinction before they ship a change.

Cover versioning strategies: URI versioning (/v1/, /v2/), header-based versioning
(Accept: application/vnd.api+json;version=2), and schema-level versioning for binary
protocols (protobuf field numbering, Avro schema evolution). Each has different tradeoffs
in discoverability, cache behavior, and operational overhead.

Cover the sunset pattern: how to deprecate an API version gracefully — deprecation headers,
sunset dates, migration guides — rather than either keeping dead code forever or breaking
consumers without warning.

The key framing: versioning is a process, not a feature. A version number in a URL does
not make an API evolvable; a disciplined process for managing backward compatibility does.

Anchor in real systems: protobuf's field numbering system (backward and forward compatible
by design — field additions are safe, removals are not), the PostgreSQL wire protocol
(stable across major versions), Google's API Improvement Proposals (AIP) as a systematic
approach to backward compatibility.

Do NOT cover: API surface design decisions (Ch 15), REST vs. RPC choice (Part III Ch 19),
branching and release strategy (Part VII Ch 50, Ch 56). This chapter is about the
mechanics of backward compatibility — not the surface design or transport choice.
```

---

## Ch 17 — Synchronous vs. Asynchronous Communication

```
This chapter covers the fundamental architectural choice between synchronous communication
(caller blocks until response is received) and asynchronous communication (caller sends
a message and continues without waiting for a response). This is not primarily a
performance decision — it is a coupling decision. Synchronous calls create temporal
coupling: the caller and callee must both be available and healthy at the same instant.

Cover the failure-mode difference clearly: in synchronous systems, a downstream failure
propagates immediately up the call chain. In asynchronous systems, the caller succeeds
regardless of whether the downstream system is available — but at the cost of eventual
consistency and the complexity of tracking whether the work was actually done.

Cover the delivery guarantee problem: at-most-once, at-least-once, and exactly-once
delivery semantics. Exactly-once delivery in distributed systems is either impossible or
extremely expensive to approximate. Most real systems accept at-least-once and design for
idempotency (covered more deeply in Part III Ch 22).

Cover the saga pattern as the mechanism for distributed transactions across async service
boundaries — and be honest about its complexity compared to a database transaction.

Cover where each belongs: synchronous for queries (the caller needs the answer to
continue), asynchronous for commands (the caller does not need to wait for completion).
The read/write asymmetry is the practical heuristic.

Anchor in real systems: HTTP/gRPC as synchronous protocols with direct failure propagation,
Kafka as an async communication backbone and the semantics it actually provides, the
saga pattern in distributed e-commerce order flows.

Do NOT cover: the initial decomposition decision (Ch 10), specific Kafka configuration
or operations (Part IX observability), API versioning (Ch 16), data ownership (Ch 18),
exactly-once delivery mechanics in depth (Part III Ch 22 covers idempotency). This chapter
establishes when to choose each model; later chapters cover the mechanics of each.
```

---

## Ch 18 — Data Ownership Boundaries

```
In a decomposed system, each piece of data has an owner: exactly one service that is the
authoritative source of truth for that data, with all other services accessing it through
the owner's interface rather than directly. This chapter covers how to assign data
ownership and what happens when you don't.

The central failure mode: the shared database. Two services querying the same PostgreSQL
table directly are coupled through the schema — any schema change requires coordinating
both services, the table's semantics diverge because neither service fully understands
the other's usage, and the resulting system is a distributed monolith (Ch 03 vocabulary)
at the data layer.

Cover the database-per-service pattern: each service owns its own schema and data store,
and exposes its data only through its API. The cost: cross-service queries that were
trivial joins become synchronous API calls or async projections. The benefit: schema
evolution and data-model changes are contained within the owning service.

Cover CQRS (Command Query Responsibility Segregation) as the mechanism for building
read-optimized projections without compromising data ownership: the owning service handles
writes; read-optimized projections are built from events emitted by the owner.

Cover the practical challenge of deciding what "owns" data when multiple services have
legitimate interest in the same domain entity (a user, an order, a product). The resolution
is usually the domain that is authoritative for the entity's lifecycle — the service that
creates, updates, and deletes it.

Anchor in real systems: the shared database anti-pattern in microservices (the primary
failure mode to dwell on), event sourcing as a data ownership mechanism where the event
log is the source of truth, CQRS projections in an order management system.

Do NOT cover: the initial decomposition decision (Ch 10), synchronous vs. async mechanics
(Ch 17), API versioning (Ch 16), database replication for read scaling (Part XII), specific
database technology choices. This chapter is about ownership boundaries — not the
technology behind them or the APIs that expose them.
```
