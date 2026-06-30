# Chapter 13 — Coupling and Cohesion at the Architecture Level

**Prerequisites:** [Part I, Ch 03 — Coupling and Cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md), [Ch 07 — Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md), [Part II, Ch 10 — Monolith vs. Service Decomposition](ch10-monolith-vs-service-decomposition.md), [Ch 12 — Dependency Direction and Inversion](ch12-dependency-direction-inversion.md). Specifically: afferent/efferent coupling, connascence, partial failure, and the distributed monolith anti-pattern.

**New vocabulary introduced:** bounded context, temporal coupling

**Key takeaways:**
- Coupling and cohesion don't change definition at the architecture level, but the stakes do: a coupling mistake between services produces network partitions and coordinated deployments instead of a local compile error, because interfaces are far harder to change once another team depends on them.
- The question this chapter answers is where service boundaries belong. The domain-driven-design answer is the bounded context — the largest scope within which a domain model's terms and rules stay internally consistent. That's architectural cohesion: everything inside shares one model; the interface to the outside is narrow and deliberate.
- The shared database is the canonical architectural-coupling failure. Two services querying the same tables are tightly coupled through the schema no matter how independently they're deployed — a single column change now requires coordinating both.
- Event-driven communication reduces temporal coupling between services, at a real cost: eventual consistency, and debugging that now has to follow an indirect, asynchronous causal chain instead of a stack trace.

---

## Bounded Contexts as Architectural Cohesion

**What it is:** A bounded context, from Eric Evans's *Domain-Driven Design*, is the architectural perimeter within which a specific domain model and its vocabulary remain internally consistent. It is high cohesion (Ch 03) applied at the scope of a whole service or subsystem, not a single module.

**Why it exists:** Large systems inevitably develop multiple, incompatible meanings for the same word. "User" in a billing context needs a credit card, a billing address, a tax ID. "User" in an authentication context needs a password hash and a second factor. Forcing both into one global entity doesn't unify the domain — it just hides the fact that two different models were coupled together by name alone.

**Options:**
1. **Global canonical model** — one enterprise-wide schema for each entity, used identically by every service
2. **Bounded contexts** — each context defines its own model in its own vocabulary, with explicit translation where contexts exchange data

**Trade-offs:**
- *Global canonical model:* removes duplication and gives everyone the same mental model up front, but couples every team to the same schema — a change billing needs forces auth to absorb a schema update that has nothing to do with authentication.
- *Bounded contexts:* maximizes team autonomy, since each service only models what it actually needs — but every boundary crossing now needs translation logic, and that logic has to be actively maintained as both sides evolve.

**When to choose each:**
- *Global canonical model:* foundational primitive types only, or systems small enough to still be a single deployable with one team.
- *Bounded contexts:* the default for any system with more than one team or more than one genuinely distinct domain.

**Common failure modes:**
- **The shared-vocabulary illusion:** two services use the same term — "Order," "Account" — and assume it means the same thing, until a field one context added breaks an assumption silently held by the other.
- **The Enterprise Service Bus hub-and-spoke:** every inter-service message is forced through a central bus into one massive canonical schema, which becomes the tightest possible coupling point — every service is now coupled to the bus's release schedule, not just to the services it actually talks to.

**Example:** In an e-commerce system, Shipping and Inventory are separate bounded contexts. They exchange a narrow, intentional event payload — `OrderPlaced` — rather than sharing one sprawling `Order` object that also carries UI rendering fields the warehouse has no use for. **[Consensus: a bounded context, not a database table or a team, is the right unit to draw a service boundary around]**

---

## The Shared Database Anti-Pattern

**What it is:** Two or more independently deployed services reading and writing the same database schema directly, instead of through each other's interfaces.

**Why it exists:** It is almost never chosen deliberately — it's the path of least resistance when a team extracts a service but doesn't want to design (or wait for) a proper data interface yet. It preserves short-term convenience at the cost of the autonomy the split was supposed to buy.

**Options:**
1. **Shared schema** — multiple services connect to the same database and query the same tables directly
2. **Strict ownership** — each service exposes its data only through its own API; no other service touches its tables

**Trade-offs:**
- *Shared schema:* trivial cross-service queries (an ordinary SQL join) and no network overhead — but it destroys deployment independence outright. Altering a column now requires a coordinated, simultaneous release across every service that touches it.
- *Strict ownership:* guarantees schema autonomy and real deployment independence, but pushes data integration out of the database and onto the network — a join becomes an API call, an async projection, or a separate analytical store (Ch 18 covers this mechanism in depth).

**When to choose each:**
- *Shared schema:* acceptable only inside a deliberate modular monolith (Ch 10), where the deployment unit really is unified.
- *Strict ownership:* a hard prerequisite the moment a service is extracted as an independently deployed unit.

**Common failure modes:**
- This is exactly how a [distributed monolith](../part01-systems-thinking/ch03-coupling-and-cohesion.md) forms at the data layer: an organization splits a monolith into a dozen services but leaves the original database intact underneath all of them, paying the full latency and operational tax of distribution while keeping every bit of the monolith's deployment coupling.
- **Read-your-neighbor's-table:** a service bypasses another service's API entirely because the data is "right there" in the shared schema, reintroducing coupling the API boundary was supposed to prevent.

**Example:** Checkout and Inventory both running `UPDATE` statements against the same `products` table will eventually corrupt each other's assumptions about what that table means. A correctly cohesive boundary instead forces Checkout to call Inventory's API to reserve stock — the schema stays entirely behind Inventory's interface, which is information hiding (Ch 04) applied at the network level. **[Strong Recommendation: database-per-service is a hard prerequisite for extraction, not an optional hardening step]**

---

## Event-Driven Decoupling

**What it is:** Services communicating by publishing and consuming events through a broker, rather than calling each other synchronously — replacing a direct call with an announcement that something happened.

**Why it exists:** A synchronous call creates **temporal coupling**: caller and callee must both be up, healthy, and reachable at the same instant — the architectural expression of the connascence of execution order described in [Ch 03](../part01-systems-thinking/ch03-coupling-and-cohesion.md). If a service blocks on a downstream call, that downstream service's availability becomes the caller's availability too. Publishing an event instead lets the publisher finish its work and respond to its own caller without waiting for every interested party to react.

**Options:**
1. **Synchronous request/response** — services call each other directly and block for the result
2. **Asynchronous event choreography** — services publish events to a broker; interested services subscribe and react independently

**Trade-offs:**
- *Synchronous:* immediate consistency and a debuggable, linear call chain, but a failure anywhere in the chain propagates immediately to the caller (Ch 07).
- *Asynchronous:* a publisher can return success to its own caller even while a downstream subscriber is completely offline, but every subscriber is now eventually, not immediately, consistent with the event — and tracing a bug means following an indirect causal chain through a broker instead of a stack.

**When to choose each:**
- *Synchronous:* queries, and any operation where the caller genuinely cannot proceed without the answer.
- *Asynchronous:* state-changing side effects and cross-domain notifications where the caller doesn't need to wait for every consumer to finish reacting.

**Common failure modes:**
- **Eventual-consistency collapse:** a user changes their password, published as an event, then immediately tries to log in — and the login fails because the auth service hasn't yet consumed the event. The system is behaving exactly as eventually-consistent systems are defined to behave; it just doesn't match what the user experiences as "done."
- **Event explosion:** every state change becomes its own event with no clear ownership of meaning, until the event stream is harder to reason about than the synchronous calls it replaced.

**Example:** When a user registers, the user service does not call the email, billing, and analytics services synchronously. It writes one `UserRegistered` event to a Kafka topic; each of those services consumes it at its own pace, fully decoupled from the user service's uptime. The delivery guarantees a broker like Kafka actually provides, and how to design for them, are covered in depth in [Ch 17](ch17-sync-vs-async-communication.md).

---

## Why Smart Engineers Disagree: Strict Autonomy vs. Operational Simplicity

The sharpest divide in boundary design is between engineers optimizing for team autonomy and engineers optimizing for operational simplicity, and it tracks organizational scale almost exactly.

Engineers in large organizations push for strict database-per-service isolation and asynchronous decoupling everywhere. They've seen a shared dependency become the thing that blocks an entire team's roadmap, and they accept the cost of eventual consistency, distributed tracing, and message brokers as the price of never having to wait on another team to deploy.

Engineers in smaller organizations argue that event-driven microservices are an accidental-complexity trap: the cognitive cost of debugging eventual consistency across a dozen bounded contexts is higher than the cost of two teams simply agreeing on a deployment window.

Conway's Law (Ch 08) resolves the disagreement more than either side's general argument does. At a few hundred engineers, communication overhead is large enough that architectural decoupling — and its complexity tax — pays for itself. At twenty engineers, that same tax is pure speculative future-proofing (Ch 05): there's no coordination cost yet to be solving for. Bounded contexts should be hardened into actual network partitions only once the organization's communication structure has grown enough to need them — not on the assumption that it eventually will.

*Concepts expanded in later chapters: messaging mechanics, ordering, and delivery guarantees (Part II, Ch 17); data ownership and the database-per-service pattern in depth (Part II, Ch 18).*
