# Chapter 17 — Synchronous vs. Asynchronous Communication

**Prerequisites:** [Part I, Ch 07 — Reliability as a Design Principle](../part1-systems-thinking/ch07-reliability-as-a-design-principle.md), [Ch 08 — Local vs. Global Optimization](../part1-systems-thinking/ch08-local-vs-global-optimization.md), [Part II, Ch 13 — Coupling and Cohesion at the Architecture Level](ch13-coupling-cohesion-architecture-level.md). Specifically: partial failure, Little's Law, and temporal coupling.

**New vocabulary introduced:** saga pattern, compensating action

**Key takeaways:**
- The choice between synchronous and asynchronous communication is not primarily about performance — it's about temporal coupling (Ch 13). A synchronous call requires caller and callee to both be healthy at the same instant; an asynchronous one doesn't.
- A synchronous chain fails by propagation: one slow or down dependency exhausts threads and connection pools all the way up the call stack. An asynchronous one fails by divergence: the caller succeeds while the system has not yet converged on the truth.
- Exactly-once delivery across a real network boundary is not achievable — only at-most-once or at-least-once are. Production systems should default to at-least-once and build idempotency into every consumer, not chase the illusion of exactly-once.
- Once a write spans more than one service, there is no database transaction to roll back. The saga pattern coordinates multi-step distributed work with explicit compensating actions instead, and it's measurably harder to reason about than a transaction — adopt it only when the workflow genuinely can't be a single service's job.

---

## The Communication Paradigm: Synchronous vs. Asynchronous

**What it is:** Whether a caller blocks its execution waiting for a downstream response (synchronous), or dispatches a message and continues immediately without waiting (asynchronous).

**Why it exists:** To deliberately manage temporal coupling and the blast radius of a downstream failure, not to optimize throughput.

**Options:**
1. **Synchronous (blocking)** — HTTP, gRPC: the caller sends a request and waits for the response
2. **Asynchronous (non-blocking)** — a message broker (Kafka, RabbitMQ): the caller writes a message and returns immediately

**Trade-offs:**
- *Synchronous:* immediate consistency and a linear, easy-to-reason-about failure path — if it fails, the caller knows right away. But a downstream slowdown propagates immediately up the call chain, and enough blocked threads waiting on one slow dependency exhausts the caller's own capacity.
- *Asynchronous:* the caller succeeds regardless of whether the downstream system is currently online — temporal coupling is eliminated, not just reduced. But the system is now eventually consistent, the UI has to poll or push to learn when work actually finishes, and tracing a bug means following an indirect, asynchronous chain instead of a stack trace.

**When to choose each:**
- *Synchronous:* queries, where the caller genuinely cannot proceed without the answer, and any mutation with a hard regulatory requirement for immediate, transactional confirmation.
- *Asynchronous:* commands, where the caller only needs the intent durably recorded — background processing, fan-out, high-throughput ingestion. This read/write asymmetry is the practical default heuristic: reads tend synchronous, writes tend asynchronous.

**Common failure modes:**
- **The cascading timeout:** Service A calls B, which calls C. C hits a database lock and takes ten seconds to respond. B's threads block waiting on C; A's threads block waiting on B. Within seconds, all three exhaust their connection pools — a localized lock in C becomes a platform-wide outage, entirely through synchronous propagation.

**Example:** An API gateway making a synchronous HTTP call to an auth service requires the auth service to be alive at that instant. An order service publishing an `OrderPlaced` event to Kafka and immediately returning HTTP 200, by contrast, is fully decoupled from whether the billing service consuming that event is online right now. **[Consensus: synchronous calls require every dependency in the chain to be healthy simultaneously; asynchronous calls trade that requirement for eventual consistency]**

---

## Message Delivery Semantics

**What it is:** The guarantee a broker makes about how many times a message is delivered: at most once, at least once, or exactly once.

**Why it exists:** Networks drop packets and consumers crash mid-processing. A system has to explicitly decide what happens when it can't tell whether a message was actually processed.

**Options:**
1. **At-most-once** — delivered once; if the consumer crashes before processing, the message is lost
2. **At-least-once** — redelivered until acknowledged; never silently lost, but can be delivered more than once
3. **Exactly-once** — delivered and processed exactly one time, regardless of failures

**Trade-offs:**
- *At-most-once:* lowest overhead, highest throughput, but guarantees data loss during a partition or a consumer redeploy.
- *At-least-once:* no data loss, but guarantees duplicates eventually arrive, pushing the burden of idempotency onto every consumer.
- *Exactly-once:* the simplest mental model for a consumer to write against — but across a real, heterogeneous network boundary it isn't actually achievable; approximating it costs the heavy coordination overhead of two-phase commit.

**When to choose each:**
- *At-most-once:* loss-tolerant telemetry and high-frequency metrics, where losing one data point is irrelevant.
- *At-least-once:* the default for essentially all production business commands and domain events.
- *Exactly-once:* never rely on it across a genuine service boundary; it's an illusion that fails the first time a partition occurs.

**Common failure modes:**
- **The non-idempotent retry:** an at-least-once broker delivers "charge credit card." The payment service charges the card, then loses power a moment before acknowledging. The broker, assuming failure, redelivers to a different node, which charges the card again. The customer is billed twice — not because anything malfunctioned, but because at-least-once delivery worked exactly as specified.

**Example:** Kafka defaults to at-least-once delivery. Experienced teams don't fight that constraint; they accept it and make consumers idempotent — commonly a unique idempotency key enforced with a database `UNIQUE` constraint — so a duplicate delivery is simply ignored on arrival. The mechanics of idempotency keys are covered in depth in [Part III, Ch 22].

---

## The Saga Pattern: Distributed Transactions

**What it is:** A way to maintain consistency across a multi-step workflow that spans several independently deployed services, using a sequence of local transactions and explicit compensating actions instead of a single atomic database transaction.

**Why it exists:** A monolith gets `BEGIN`/`COMMIT`/`ROLLBACK` from its database for free. Once that workflow is split across services, there's no longer a single transaction to roll back — if step three of five fails, something has to explicitly undo the effects of steps one and two.

**Options:**
1. **Choreography** — services publish domain events; other services subscribe, do their own local work, and publish their own events, with no central coordinator
2. **Orchestration** — a dedicated orchestrator sends explicit commands to each service in turn, tracks the result, and decides the next step

**Trade-offs:**
- *Choreography:* no central bottleneck or single point of failure, and each team stays autonomous — but the overall business process becomes implicit, spread across however many services react to however many events, with no single place to answer "what state is order #123 actually in?"
- *Orchestration:* one observable state machine for the entire transaction, easy to trace and reason about — but the orchestrator becomes a coupling point that every workflow change has to go through, and a bottleneck for the team that owns it.

**When to choose each:**
- *Choreography:* short, two- or three-step flows with few failure states, or reacting to an event that originates outside your own bounded context anyway.
- *Orchestration:* complex, multi-step business processes — order fulfillment, onboarding — where knowing the exact state of an in-flight transaction is an actual operational requirement.

**Common failure modes:**
- **The missing compensating action:** a saga has no automatic rollback. If an engineer forgets to write the compensating action for a step, or that action itself fails without being routed to a dead-letter queue for manual recovery, the system is left in a permanently half-committed state with no path back to consistency.

**Example:** An orchestrated checkout saga: the orchestrator creates a `Pending` order, tells the payment service to charge the card (succeeds), then tells inventory to reserve the item (fails — out of stock). Because the transaction can't complete, the orchestrator runs the compensating actions: payment is told to refund the charge, and the order is marked `Failed`. Nothing rolled back automatically — every step of the undo was its own explicit command. **[Strong Recommendation: orchestration for any workflow where "what state is this transaction in right now" is a question the business will actually ask]**

---

## Why Smart Engineers Disagree: The Illusion of Exactly-Once

The sharpest disagreement in asynchronous architecture is whether exactly-once delivery is a legitimate design target or a trap.

Engineers working inside closed ecosystems — Kafka Streams, Flink — point to transactional producer/consumer APIs that genuinely guarantee a message is read, processed, and written to an output topic exactly once, entirely within that ecosystem. They argue that pushing this guarantee into the infrastructure frees product engineers from writing idempotency and compensation logic by hand.

Engineers who work across heterogeneous boundaries — most SREs — treat exactly-once as a marketing claim that collapses the moment it matters. A closed system can guarantee exactly-once *internal* state transitions, but the instant that system has to send an email, call a payment provider, or write to an external database, the guarantee evaporates: the network only ever lets you choose between losing a message and sending it twice.

The pragmatic resolution: exactly-once is real but local — it holds only inside a single, closed, transactional system, and stops holding the moment a message crosses into anything heterogeneous, which is most of a real distributed system's surface. Rather than chase a guarantee that only sometimes applies, build every asynchronous consumer to be idempotent by default, and treat the rare cases of true internal exactly-once as a bonus, not the foundation the architecture depends on.

*Concepts expanded in later chapters: data ownership and the database-per-service pattern (Part II, Ch 18); idempotency key mechanics in depth (Part III, Ch 22); Kafka operational configuration (Part IX).*
