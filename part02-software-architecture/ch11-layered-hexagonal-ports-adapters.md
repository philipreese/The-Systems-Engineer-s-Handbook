# Chapter 11 — Layered, Hexagonal, and Ports-and-Adapters Architecture

**Prerequisites:** [Part I, Ch 03 — Coupling and Cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md), [Ch 04 — Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Part II, Ch 10 — Monolith vs. Service Decomposition](ch10-monolith-vs-service-decomposition.md). Specifically: afferent/efferent coupling, the distinction between encapsulation and information hiding, and connascence.

**New vocabulary introduced:** hexagonal architecture (ports-and-adapters), layer leakage, Data Transfer Object (DTO)

**Key takeaways:**
- Chapter 10 decided whether to split a system into deployable units. This chapter is about the unit itself: once you have a service, how should the code inside it be organized? The answer in every case is a decision about dependency direction, not file layout.
- Layered architecture lets dependencies flow downward through presentation, business logic, and data access. It's intuitive and fast to build, and it fails predictably: infrastructure concepts leak upward through the layers as the system ages.
- Hexagonal architecture inverts that: business logic sits at the center, defines the interfaces ("ports") it needs, and infrastructure implements them from the outside ("adapters"). This is information hiding (Ch 04) applied at the architectural level — the core never knows it's talking to PostgreSQL or Kafka.
- Hexagonal isolation is not free. It buys fast, infrastructure-free unit tests and the ability to change infrastructure without touching domain logic, at the cost of interface boilerplate and a mapping layer at every boundary. The right choice depends on how much real business logic exists to protect.

---

## Layered (N-Tier) Architecture

**What it is:** Code organized into horizontal tiers — typically presentation, business logic, and data access — where each layer depends only on the layer immediately below it.

**Why it exists:** It maps directly onto how a request actually moves through a system: a controller receives it, a service handles it, a repository persists it. It's intuitive, easy to teach, and requires no upfront interface design.

**Options:**
1. **Strict layering** — a layer may only call the layer directly below it
2. **Relaxed layering** — higher layers may bypass intermediate layers for convenience (e.g., a controller reading directly from a repository for a simple query)
3. **Layered in name only** — layers exist as a file-organization convention with no enforced dependency rule

**Trade-offs:**
- *Strict layering:* predictable, traceable control flow, but forces redundant pass-through methods in the business layer for trivial reads, adding accidental complexity (Ch 02) without adding meaning.
- *Relaxed layering:* faster for simple operations, but every bypass is also a bypass of whatever validation or authorization logic lived in the skipped layer.
- *Name-only layering:* fastest short-term, but with no enforced rule the layers collapse into a dependency tangle as soon as deadline pressure arrives.

**When to choose each:**
- *Strict layering:* the default for systems of moderate complexity where the rule can actually be enforced (linting, module boundaries, code review).
- *Relaxed layering:* acceptable for read-only, non-authoritative queries where the bypassed layer truly has nothing to add.
- *Name-only:* prototypes and throwaway systems only.

**Common failure modes:**
- **Layer leakage:** because the dependency points downward, the business layer is structurally exposed to the data layer's volatility. SQL exceptions surface in the API layer; ORM annotations creep into domain objects. The information hiding the layering was supposed to provide quietly stops existing.
- **The fat service layer:** all meaningful logic collapses into one layer because no one enforced where decisions belong, and that layer becomes the system's de facto god object.
- Circular dependencies between layers as ad hoc shortcuts accumulate.

**Example:** Django's MTV pattern is a pragmatic layered architecture — the View handles the request, the Model holds both persistence and a fair amount of business logic, and the Template renders the response. It is optimized for velocity and accepts, as a deliberate trade, that business logic and data access stay intertwined; in large Django codebases, that's exactly where layer leakage tends to show up first, as logic drifts into views or models depending on whoever is under the most deadline pressure.

---

## Hexagonal Architecture (Ports-and-Adapters)

**What it is:** Introduced by Alistair Cockburn in 2005. Business logic — the core — sits at the center and defines interfaces ("ports") for everything it needs from the outside world. Infrastructure connects to the core by implementing those interfaces ("adapters"). Dependencies always point inward, toward the domain.

**Why it exists:** Infrastructure changes faster, and for different reasons, than business logic does. Hexagonal architecture exists to stop that volatility from contaminating the domain — it is information hiding (Ch 04) applied at the architecture level, not just the module level.

**Options:**
1. **Driving (primary) adapters** — infrastructure that initiates action into the core: a REST controller, a CLI command, an event listener
2. **Driven (secondary) adapters** — infrastructure the core uses to produce side effects: a PostgreSQL repository, a payment gateway client
3. **Selective hexagonal** — ports only around the boundaries that are actually volatile, direct calls elsewhere

**Trade-offs:**
- *Full hexagonal isolation:* forces the domain to be framework-agnostic and lets the entire business rule set be unit-tested in milliseconds against in-memory fakes, with no database or network involved. But it requires defining and maintaining an explicit interface for every external interaction, plus the dependency-injection wiring to satisfy them.
- *Selective hexagonal:* lower upfront cost, but only protects the boundaries someone correctly predicted would be volatile — protection for a boundary that turns out to matter later has to be retrofitted.
- *Infrastructure-first (no ports):* cheapest to start, but the domain is coupled to a framework and a database from day one, and migrating either later means touching business logic, not just an adapter.

**When to choose each:**
- *Full hexagonal:* domain logic with real rules that will outlive the current database or delivery framework — payment systems, pricing engines, anything with compounding business rules worth testing in isolation.
- *Selective/none:* simple data-entry services and ephemeral microservices where the database effectively *is* the domain, and there's no meaningful logic to protect from it.

**Common failure modes:**
- **The empty core:** a team builds the full port/adapter ceremony for a service that just writes JSON payloads to a table. The architecture adds indirection and buys zero isolation value, because there was never any volatile business logic to isolate in the first place.
- **Leakage through adapters:** infrastructure-specific shortcuts (an ORM entity passed straight through, a vendor-specific error code checked in domain code) defeat the isolation the ports were meant to provide.
- **False independence:** assuming the system is now portable to a different database or message broker because adapters exist, when the domain's behavioral assumptions are still implicitly shaped by the original infrastructure's semantics.

**Example:** A billing service originally called `StripeClient` directly from `BillingService`. Restructured into hexagonal form, `BillingService` calls a `PaymentProcessor` port; `StripeClient` becomes one adapter implementing it. The core never changes if the company adds or replaces a payment provider — only a new adapter is written. Spring Boot applications illustrate both ends of this spectrum in the same ecosystem: most default to a layered configuration with services calling repositories directly, while teams building long-lived core domains wire dependency injection to enforce a strict hexagonal boundary instead. **[Strong Recommendation: invest in ports where infrastructure churn or test cost is real; skip them where the database is the entire application]**

---

## Shared Models vs. Strict Boundary Mapping

**What it is:** The decision of whether a single data structure travels unchanged from the database through business logic and out to the network, or whether each boundary translates data into a structure of its own.

**Why it exists:** Data has to move through every layer or boundary in the system. Whether that movement happens through one shared structure or through explicit translation at each crossing determines how tightly the layers are coupled to each other's shape.

**Options:**
1. **Shared model** — one class represents the data end-to-end: queried from the database, passed through business logic, serialized directly into the response
2. **Strict boundary mapping** — each boundary owns its own shape (an ORM entity, a domain model, an API Data Transfer Object (DTO)) with explicit mapping between them

**Trade-offs:**
- *Shared model:* eliminates mapping boilerplate and is fast to write, but creates connascence of type and value (Ch 03) straight through the system — a database column rename instantly changes the JSON the API returns.
- *Strict boundary mapping:* the API contract is mechanically decoupled from the schema, so either can change without forcing a change in the other, but it costs real mapper code and the memory overhead of duplicating data at every boundary.

**When to choose each:**
- *Shared model:* layered architectures with a single, tightly coupled internal consumer, where rapid delivery matters more than contract stability.
- *Strict boundary mapping:* hexagonal architectures, and any service with a published API (Ch 15) — an accidental contract change here cascades to every external consumer.

**Common failure modes:**
- **The god model:** a shared `User` class accumulates ORM annotations, JSON serialization tags, GraphQL directives, and core validation logic simultaneously, until a change meant to support one new UI field breaks the database write path.

**Example:** In a hexagonal REST API, a request arrives as a `CreateUserRequest` DTO. The driving adapter maps it to a plain `User` domain model and passes it through the port into the core, which applies its rules. The core hands the domain model to a driven repository adapter, which maps it again into a `UserEntity` for the PostgreSQL insert. Three distinct shapes, three explicit mappings, zero structural coupling between the database schema and the API contract.

---

## Why Smart Engineers Disagree: Indirection vs. Simplicity

The most persistent disagreement at this layer of architecture is between isolation purists and pragmatic minimalists, and it comes down to how each side prices the probability that infrastructure will actually change.

Engineers optimizing for longevity and testability treat the framework and the database as hostile, volatile dependencies. They accept the tax of ports, adapters, and mapping layers because it buys a domain that can be unit-tested in milliseconds, with no database or container required, and that stays untouched when infrastructure changes around it.

Engineers optimizing for delivery speed argue that the database swap hexagonal architecture protects against almost never happens — mature systems rarely migrate from PostgreSQL to MongoDB — and that writing dozens of interfaces to guard against a migration that will never occur is future-proofing (Ch 05) by another name.

Both are right under different conditions, and the deciding factor is the essential complexity (Ch 02) of the business logic itself. A data-entry tool whose job is moving forms into tables has no domain to protect — the database *is* the domain, and hexagonal structure there is pure accidental complexity. A pricing engine with hundreds of compounding, overlapping rules is the opposite case: testing those rules against a live database makes the suite slow and brittle, and the real payoff of hexagonal architecture isn't theoretical database portability — it's running thousands of business-rule tests in sub-millisecond isolation, every day, for as long as the system lives.

*Concepts expanded in later chapters: dependency inversion as a formal principle (Part II, Ch 12), API surface design at the boundary (Part II, Ch 15), module and file structure within a layer (Part IV, Ch 27).*
