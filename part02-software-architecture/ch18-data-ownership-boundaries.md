# Chapter 18 — Data Ownership Boundaries

**Prerequisites:** [Part I, Ch 03 — Coupling and Cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md), [Part II, Ch 10 — Monolith vs. Service Decomposition](ch10-monolith-vs-service-decomposition.md), [Ch 13 — Coupling and Cohesion at the Architecture Level](ch13-coupling-cohesion-architecture-level.md), [Ch 17 — Synchronous vs. Asynchronous Communication](ch17-sync-vs-async-communication.md). Specifically: connascence of schema and value, the distributed monolith anti-pattern, and temporal coupling.

**New vocabulary introduced:** database-per-service pattern, CQRS (Command Query Responsibility Segregation)

**Key takeaways:**
- In a decomposed system, boundaries are defined by who controls the data, not by where the code lives. Every piece of mutable data needs exactly one authoritative owner; everyone else gets a read-only view through that owner's API, never direct access to its storage.
- The shared database is the central failure mode of this chapter, as it is of Ch 13: two services querying the same tables are coupled by schema regardless of how independently they're deployed, and that coupling shows up as a data-layer distributed monolith.
- When multiple services have a legitimate interest in the same entity, ownership goes to whichever service controls its lifecycle — creation, valid state transitions, deletion — not to whichever service uses it most.
- Database-per-service breaks the trivial SQL join. CQRS is the mechanism for getting a fast, cross-domain read back without giving any other service write access to data it doesn't own: the owner emits events, and downstream services build their own local, eventually consistent projections.

---

## State Isolation: The Database-per-Service Pattern

**What it is:** Each service's persistence layer — schema, connection string, storage engine — is reachable only by that service. No other service connects to it directly; everything goes through the owner's API.

**Why it exists:** To prevent the shared database, the most direct route to architectural coupling. Two services querying identical tables are coupled by connascence of schema (Ch 03) even when they're deployed completely independently — a column type change in one instantly breaks the other.

**Options:**
1. **Shared database schema** — multiple services connect to the same database and query the same tables directly
2. **Database-per-service** — each service owns its datastore exclusively; everything else goes through its API

**Trade-offs:**
- *Shared schema:* cross-domain data is a trivial, fast SQL join — but deployment autonomy is gone, because neither service fully understands the other's query patterns, so a column rename now needs a coordinated cross-team release.
- *Database-per-service:* deployment isolation and schema autonomy are guaranteed — the owner could migrate from PostgreSQL to MongoDB without telling anyone — but a cross-domain query that used to be a join is now an API call or async projection, with the latency and partial-failure cost that implies.

**When to choose each:**
- *Shared schema:* only inside a deliberate modular monolith (Ch 10), where the whole thing deploys as one unit anyway.
- *Database-per-service:* the hard prerequisite the moment a real process boundary exists.

**Common failure modes:**
- **The data-layer distributed monolith:** an organization splits into fifteen microservices for velocity, but leaves all of them connected to the same legacy PostgreSQL database. Table semantics drift as different teams overload the same columns for different purposes, and the shared database becomes a fragile central bottleneck — every operational cost of microservices, none of the autonomy.

**Example:** In a properly decoupled architecture, the billing service owns the `invoices` table. If support needs a user's invoice history, it doesn't run `SELECT * FROM invoices` — it calls `GET /billing/users/{id}/invoices`. Billing is the absolute gatekeeper to its own data. **[Strong Recommendation: database-per-service is a hard prerequisite for extraction, not a hardening step to get to eventually]**

---

## Lifecycle-Based Ownership

**What it is:** The rule for resolving disputes when more than one service has a legitimate interest in the same entity — a "User," an "Order," a "Product."

**Why it exists:** Ownership is rarely obvious for entities multiple domains genuinely care about. Authentication, profile, and billing services all need data about a "User." Without an explicit rule, more than one of them will try to validate and mutate the same entity, and their assumptions will diverge.

**Options:**
1. **Multiple writers** — any service that needs the entity can mutate it directly
2. **Lifecycle authority** — one service, the one that controls creation, valid state transitions, and deletion, is the sole owner; everyone else holds a read-only reference

**Trade-offs:**
- *Multiple writers:* any team can build features touching the entity without waiting on another team's API roadmap — but validation rules duplicate and drift, until one service accepts a state the other considers invalid.
- *Lifecycle authority:* guarantees one source of truth and one place business rules are enforced — but every other service now has to ask the owner to make a change instead of just making it.

**When to choose each:**
- *Lifecycle authority:* the default, without exception, for any entity crossing a service boundary.
- *Multiple writers:* never, for a core domain entity shared across services — it isn't a real option, it's the failure mode this section exists to name.

**Common failure modes:**
- **The split-brain entity:** the auth service updates a user's email in its own database, but the marketing service keeps a separate `users` table that's never notified. Password resets now go to the new address; promotional email still goes to the old one. The entity's state has fractured across the architecture with no single version that's actually correct.

**Example:** In an order management system, the cart service creates a cart, but once checkout completes, the order service becomes the sole lifecycle authority for the `Order`. The fulfillment service physically ships the package, but it doesn't own the order entity — it holds the `order_id` as a reference and dispatches an `UpdateOrderStatus` command back to the order service to actually change the authoritative state. **[Consensus: ownership follows lifecycle control, not usage frequency]**

---

## Cross-Boundary Queries: CQRS and Projections

**What it is:** Command Query Responsibility Segregation (CQRS) separates the model used to write data from the model used to read it. Applied across service boundaries, it's the mechanism for building a fast, local, read-optimized view of data another service owns.

**Why it exists:** Database-per-service breaks the SQL join. A page combining data from a user service, an order service, and a catalog service can't make three synchronous calls in real time without risking the latency SLO — and an N+1 fan-out across service boundaries is far more expensive than the same fan-out inside a database.

**Options:**
1. **Synchronous API composition** — a gateway calls each owning service in real time and joins the responses in memory
2. **Asynchronous CQRS projections** — the owner publishes domain events; downstream services consume them and build their own local, read-optimized tables

**Trade-offs:**
- *API composition:* conceptually simple and always immediately consistent, but creates real temporal coupling — the query fails the moment any one upstream service is down — and the latency tax compounds with every service added to the composition.
- *CQRS projections:* fast local reads with no temporal coupling at all — the downstream service can be queried even while the owner is offline — but the projection is now eventually consistent, lagging the source of truth by however long the event pipeline takes, and the broker and event-handling infrastructure is real operational weight.

**When to choose each:**
- *API composition:* low-volume internal dashboards or anywhere immediate consistency is a hard regulatory requirement.
- *CQRS projections:* high-traffic, read-heavy, cross-domain interfaces where latency and availability matter more than read freshness.

**Common failure modes:**
- **The stale projection:** the event broker partitions, or the consumer crashes, and a search service stops receiving `ProductUpdated` events. Its index falls hours behind the authoritative catalog, and customers buy items the product service already knows are out of stock — the projection is serving data that was true, just not anymore.

**Example:** A product service owns its catalog as an immutable event log (`ProductCreated`, `PriceChanged`). A search service never touches the product service's database — it subscribes to that event stream and builds its own Elasticsearch index from it. The search service owns its projection; the product service definitively owns the data.

---

## Why Smart Engineers Disagree: Data Ownership vs. Analytical Velocity

The most persistent conflict over data ownership isn't between two architectural camps — it's between software engineers and data engineers, optimizing for genuinely different workloads.

Software engineers, focused on transactional (OLTP) correctness, defend database-per-service without compromise: it's the only thing that protects schema evolvability and transactional integrity in the live product, and a shared database is the worst anti-pattern available to them.

Data engineers and analysts, focused on analytical (OLAP) queries, see the same boundary as an organizational disaster. A question as basic as "which products are most popular with enterprise users" now requires brittle ETL pipelines pulling data back out of a dozen APIs instead of one SQL join across two tables.

Both are right about their own workload, and the resolution isn't to compromise the operational boundary — it's to stop trying to answer both questions with one system. Operational databases keep strict database-per-service isolation to protect the live application. Those same services emit change-data-capture or domain events into a centralized data warehouse, which becomes a single, shared, read-only projection built specifically for analytical queries. The operational boundary stays intact; the analytical question gets its own system to be asked in, instead of a join that would have put production uptime at risk to answer it.
