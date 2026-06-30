# Chapter 12 — Dependency Direction and Inversion

**Prerequisites:** [Part I, Ch 02 — Complexity Is the Enemy](../part1-systems-thinking/ch02-complexity-is-the-enemy.md), [Ch 03 — Coupling and Cohesion](../part1-systems-thinking/ch03-coupling-and-cohesion.md), [Ch 04 — Abstraction and Information Hiding](../part1-systems-thinking/ch04-abstraction-and-information-hiding.md), [Part II, Ch 11 — Layered, Hexagonal, and Ports-and-Adapters Architecture](ch11-layered-hexagonal-ports-adapters.md). Specifically: afferent/efferent coupling, essential vs. accidental complexity, and the ports/adapters vocabulary.

**New vocabulary introduced:** Dependency Inversion Principle (DIP), repository pattern

**Key takeaways:**
- A source code dependency is a vector: if Module A imports Module B, change flows from B into A. Chapter 11 named the patterns that control this; this chapter is about the underlying mechanism every one of those patterns depends on — the direction of the dependency arrow itself.
- The Dependency Inversion Principle (DIP) says high-level policy must not depend on low-level detail. Both should depend on an abstraction, and that abstraction must be owned by the high-level (stable) side — not by the infrastructure it's meant to isolate.
- Defining an interface is not the same as inverting a dependency. If the core programs against an interface shaped by the vendor's vocabulary, the dependency still points outward. Inversion requires the stable module to own the interface in its own vocabulary, and the infrastructure to adapt to it.
- The practical test for whether inversion is real: can the dependency be swapped — for a different vendor, or for an in-memory test double — without touching a line of core logic? The value of that test is rarely about ever actually swapping vendors; it's about running thousands of business-rule tests in milliseconds against fakes instead of a live database.

---

## Direct vs. Inverted Dependencies

**What it is:** The structural choice between a high-level module directly importing a low-level module's concrete implementation, versus the high-level module defining an interface that the low-level module implements, with the two wired together by something outside both of them.

**Why it exists:** High-level policy — how interest compounds, how an order total is calculated — is comparatively stable. Low-level detail — how the result gets serialized over a TCP socket into PostgreSQL — is comparatively volatile. If the policy module imports the database driver directly, the policy becomes coupled to the volatility of the driver, even though nothing about the policy itself changed.

**Options:**
1. **Direct dependency** — business logic imports the concrete infrastructure module directly
2. **Inverted dependency (DIP)** — business logic defines an interface; infrastructure implements it; a third party (a DI container, a `main` function) wires the two together at runtime

**Trade-offs:**
- *Direct dependency:* trivial to write and trace, minimal code volume — but it fuses business logic to a specific vendor, and isolated unit testing without a live database becomes effectively impossible.
- *Inverted dependency:* the business logic can be unit-tested in microseconds against an in-memory fake, and a vendor change never requires touching domain code — but it costs an explicit interface, a wiring mechanism, and mapping code between domain and infrastructure shapes.

**When to choose each:**
- *Direct dependency:* shell scripts, throwaway migration tools, and standard-library-level utilities with effectively zero volatility.
- *Inverted dependency:* the default for core business rules, domain entities, and anything that represents the actual value the software exists to deliver.

**Common failure modes:**
- **The transitive ripple:** a deeply embedded direct dependency changes its signature, and because dependencies point downward, every layer above it must change, retest, and redeploy to absorb a change in a low-level library that has nothing to do with the business rule that layer implements.
- **Framework leakage:** domain logic importing ORM types, HTTP types, or a cloud SDK's types directly, so the domain model is partly written in someone else's vocabulary.

**Example:** Go's `database/sql` package exposes a concrete `*sql.DB`. Passed straight into a `UserService`, that service is now structurally coupled to SQL. Inverted, the `UserService` instead depends on a `UserRepository` interface it defines itself (`FindUser(id string) User`); a `PostgresUserRepository` in the infrastructure layer implements it. Switching `PostgresUserRepository` for an in-memory fake in tests, or for a different database in production, never requires changing `UserService`. That swap — with zero change to the core — is the practical test of whether inversion is real or only cosmetic.

---

## Interface Ownership

**What it is:** The question of which side of a dependency gets to define the interface's vocabulary — its method names, its data shapes, its error types.

**Why it exists:** Defining *an* interface is not sufficient for inversion. If the core programs against an interface shaped by the infrastructure vendor — its request objects, its exception types — the dependency still points outward in everything but name. Real inversion requires the stable side to own the interface.

**Options:**
1. **Callee-owned (foreign) interface** — the core programs against an interface defined by the external library or vendor
2. **Caller-owned (local) interface** — the core defines its own interface in pure domain vocabulary; infrastructure writes an adapter that translates into it

**Trade-offs:**
- *Callee-owned:* no adapter layer to write — vendor SDK objects can be passed straight through the codebase — but vendor vocabulary (exception types, configuration objects) leaks directly into the domain.
- *Caller-owned:* the domain model stays pure and speaks only the language of the business, but it demands sustained discipline: every new feature touches both the core interface and the adapter's translation logic.

**When to choose each:**
- *Callee-owned:* acceptable only for foundational language primitives and standard interfaces (a language's built-in reader/writer abstractions, a standard HTTP client interface) — things that are themselves stable by design.
- *Caller-owned:* the default for external infrastructure, SaaS vendors, databases, and message queues — anything actually volatile.

**Common failure modes:**
- **The leaky SDK:** a team believes it has abstracted its cloud storage because it programs against the vendor's `S3Client` interface instead of the concrete class — but the interface still demands vendor-specific request structs and throws vendor-specific exceptions. Interface ownership never left the vendor. When the team tries to migrate providers, it discovers the core domain has to be rewritten anyway, because the vocabulary was AWS's all along.

**Example:** A payroll system does not call `stripe.Charge.create()` from its core. It defines its own `PaymentGateway.Process(amount Money) error` interface, in payroll vocabulary, and requires the infrastructure layer to provide a `StripeAdapter` that implements it — catching Stripe's specific HTTP errors and translating them into the domain's own error types. **[Strong Recommendation: own the interface on the stable side for anything actually volatile; importing a vendor's interface is not inversion, even when it looks like one]**

---

## The Dependency Rule: Infrastructure at the Edges

**What it is:** Robert C. Martin's framing, from *Clean Architecture*: source code dependencies must always point inward, toward higher-level policy. Concrete infrastructure — the UI, the database driver, external clients — lives at the outer edge; abstract interfaces live at the core.

**Why it exists:** It generalizes interface ownership into a system-wide rule. The business's core rules change rarely; the delivery mechanism and the storage technology change often. Forcing every dependency to point inward guarantees that a change to the edge — a new web framework, a new database — never requires the core to even be recompiled.

**Options:**
1. **Entangled infrastructure** — core logic and infrastructure logic mixed in the same functions or files
2. **Edge-isolated infrastructure** — the system is organized in concentric rings: domain at the center, use cases around it, frameworks and drivers at the outer edge, with every dependency pointing inward

**Trade-offs:**
- *Entangled infrastructure:* fast to write for a single-purpose service, but testing the business rule means testing the network and database calls along with it.
- *Edge-isolated infrastructure:* the delivery mechanism (REST to gRPC) or the storage technology can change without touching the core, but a single feature now touches more files — the edge adapter, the use case, the domain interface — than a tangled version would.

**When to choose each:**
- *Entangled:* short-lived scripts, small bounded functions, and genuine throwaway prototypes.
- *Edge-isolated:* anything expected to survive multiple years, outlive its original authors, or absorb real business-rule evolution.

**Common failure modes:**
- **The untestable core:** a tax-calculation engine embeds direct calls to a third-party currency API inside its calculation loop. The dependency points outward, toward a volatile network service, so the core's own logic can no longer be tested without the network call succeeding — and a rate limit or an outage on the third party now fails a test suite that has nothing to do with it.

**Example:** PostgreSQL sits entirely at the edge of the architecture. The core defines an `OrderStore` interface and knows nothing about SQL or row locking. At the edge, an `SQLOrderRepository` imports the Postgres driver, builds the query, executes it, and maps the relational result into a plain `Order` domain object before handing it back to the core. **[Consensus: source dependencies point toward policy, never away from it]**

---

## Why Smart Engineers Disagree: The "YAGNI" of Swappable Databases

The most common pushback against dependency inversion is YAGNI — You Aren't Gonna Need It.

The pragmatic engineer argues, correctly, that the company has never once swapped its primary relational database for a different vendor, and likely never will. Building a repository interface to pretend PostgreSQL is swappable is accidental complexity (Ch 02) paid for a migration that isn't coming — and it forecloses vendor-specific features, like Postgres's `JSONB` indexing or `RETURNING` clauses, that a direct dependency would let the team use freely.

The systems architect's case for inversion does not actually rest on that migration ever happening. The real value of inverting the dependency is the ability to swap the database for an in-memory fake *today* — running ten thousand business-rule tests in well under a second, instead of against a live database — and the cognitive firewall it builds: reading the core, an engineer reasons only about domain constraints, never about connection pooling or query semantics bleeding in from below.

Both sides are reasoning about real costs. The pragmatic engineer is right that production vendor swaps are a myth almost everywhere; the architect is right that the interface still earns its keep, just for a different reason than the one usually advertised. The resolution is to stop justifying inversion by a hypothetical vendor migration and start justifying it by the test isolation and vocabulary purity it buys daily — which also clarifies when to skip it: a module with no meaningful test surface and no vocabulary worth protecting gets nothing from an interface it will never need to satisfy twice.

*Concepts expanded in later chapters: the specific architecture patterns this principle is embedded in — hexagonal and layered (Part II, Ch 11); module and file structure for organizing the resulting interfaces (Part IV, Ch 27).*
