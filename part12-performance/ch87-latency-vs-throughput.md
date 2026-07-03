# Ch 87 — Latency vs. Throughput Trade-offs

**Prerequisites:** [Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md) (Little's Law, Theory of Constraints), [Error Budgets and SLOs](../part09-observability/ch73-error-budgets-and-slos.md) (SLI vocabulary), [Profiling-First](ch86-profiling-first.md)

**New vocabulary introduced:** tail latency, request coalescing

**Key takeaways:**
- Latency is the time to complete one unit of work; throughput is units of work completed per unit of time. "Make it faster" is ambiguous between the two, and a change that improves one routinely costs the other — this ambiguity, not a technical disagreement, is the source of most confused performance discussions.
- [Strong Recommendation] Batching is the primary throughput-over-latency lever: it amortizes a fixed per-operation cost across many operations, at the direct cost of the earliest item in a batch waiting for the batch to fill. Little's Law and the Theory of Constraints (Ch 08) already explain why this changes system capacity; that mechanism is not re-derived here.
- Tail latency is a distinct concern from average latency, not a more extreme version of the same number. A system tuned against a mean or median can look excellent on that number while performing badly at the tail — Ch 73's percentile SLI is the already-established, correct way to express this.
- Connection pooling, request coalescing, bulk database writes, and Nagle's algorithm are all the same amortization principle applied at a different layer — a scarce resource, duplicate work, a transaction boundary, or a network packet, respectively.

---

"Make this faster" is a request with two different, sometimes opposite, answers. Latency is the time a single unit of work takes to complete. Throughput is the volume of work a system completes per unit of time. A service that responds in 20 milliseconds has low latency; a service that processes a million requests per second has high throughput. Neither implies the other, and treating them as interchangeable is how teams end up optimizing the wrong number — reporting an improved requests-per-second figure to users who are complaining that their individual request is slow.

### Decision: Batch Work to Amortize Fixed Costs, or Process Immediately

**What it is:** Whether units of work are accumulated and processed together to spread a fixed per-operation cost across all of them, or each unit is processed the moment it arrives.

**Why it exists:** Most systems carry real fixed costs independent of the size of a single operation — a network round trip, a system call, a disk seek, transaction setup, lock acquisition. Immediate processing pays that fixed cost once per unit of work. Batching pays it once per batch. Ch 08 already supplies the reason this changes system-level behavior, not just per-operation cost: batching changes arrival and service-time patterns that Little's Law and the Theory of Constraints govern, and is not re-derived here.

**Options:**
- **Immediate processing** — handle each unit of work the moment it arrives.
- **Batching** — accumulate units until a size or time threshold is met, then process together.

**Trade-offs:** Immediate processing minimizes the wait for any single unit but repeats the fixed cost on every one — the earliest possible response, at the lowest possible throughput per unit of fixed overhead. Batching amortizes that fixed cost across many units, raising sustained throughput substantially, at the direct cost of the earliest items in the batch waiting for it to fill — a cost that grows if the batch is sized for throughput without a time-based ceiling.

**When to choose each:** Immediate processing where a hard per-request latency deadline exists — interactive sessions, anything a human is waiting on synchronously. Batching where data volume exceeds continuous processing capacity, or where the fixed cost being amortized (a disk sync, a transaction commit, a network round trip) is large relative to the actual work — bulk ingestion, analytics pipelines, database writes.

**Common failure modes:** A team building a telemetry ingestion pipeline commits each incoming record as its own database transaction; write-ahead-log fsync and transaction setup dominate every write, and the storage engine spends most of its cycles on lock and journal overhead rather than writing data. The same records committed in a single bulk operation amortize that fixed cost across thousands of rows at once.

**Example:** PostgreSQL demonstrates the gap directly: thousands of individual `INSERT` statements exhaust disk write capacity through repeated transaction and fsync overhead, while a multi-row insert or a `COPY` block amortizes that overhead across the whole batch and drives throughput close to the hardware's raw limit. Kafka goes further and makes this trade-off the center of its design: log-structured, append-only storage and configurable batching (`linger.ms`) allow extremely high sustained write throughput, at the cost of individual messages waiting briefly before being flushed. Kafka does not hide this latency cost — it is the deliberate, stated trade for throughput at the system's design center.

### Decision: Batch Network Writes (Nagle's Algorithm), or Send Immediately (`TCP_NODELAY`)

**What it is:** Whether small outgoing TCP writes are held and consolidated into larger packets before transmission (Nagle's algorithm, enabled by default), or sent to the wire the instant the application writes them (`TCP_NODELAY`).

**Why it exists:** TCP/IP framing carries real fixed overhead — a minimum of roughly 40 bytes of header per packet, independent of payload size. A stream of small writes sent individually pays that overhead on every one; consolidating them amortizes it, at the cost of delaying each write until enough data has accumulated or a prior write has been acknowledged.

**Options:**
- **Nagle's algorithm (default)** — buffer small writes until a full segment accumulates or an outstanding write is acknowledged.
- **`TCP_NODELAY`** — disable buffering; transmit every write immediately.

**Trade-offs:** Nagle's algorithm maximizes bandwidth efficiency and reduces the fraction of the wire spent on header overhead, at the cost of inflating the latency of small, latency-sensitive writes waiting in the local buffer. `TCP_NODELAY` delivers the lowest possible per-write latency, at the cost of flooding the network stack with many small packets under high write frequency, which can shift the bottleneck to context-switch and interrupt overhead instead.

**When to choose each:** [Legitimate Trade-off] `TCP_NODELAY` for interactive, latency-sensitive protocols — SSH sessions, multiplayer game state, RPC frameworks, and in-memory stores like Redis where sub-millisecond responsiveness is the point. Nagle's default behavior for bulk transfer and background replication, where sustained bandwidth efficiency matters more than any single write's latency.

**Common failure modes:** *The delayed-ACK deadlock.* Nagle's algorithm on the sender, combined with TCP delayed acknowledgment on the receiver, produces a structural stall: the sender waits for more data before transmitting, while the receiver waits before acknowledging what it already has. The interaction imposes a latency floor — commonly cited around 200 ms — on small request-response exchanges, and it reproduces intermittently enough that it is usually diagnosed as network flakiness before anyone identifies the actual interaction.

**Example:** Interactive remote shells disable Nagle's algorithm so individual keystrokes appear immediately; bulk file transfer tools leave it enabled because aggregate bandwidth, not per-packet latency, is what the transfer is measured on.

### Decision: Amortize a Scarce Resource or Duplicate Work — Connection Pooling and Request Coalescing

**What it is:** Two related throughput levers that trade a small amount of queueing latency for eliminating repeated cost: connection pooling reuses an already-established connection instead of paying setup cost per request; request coalescing merges multiple concurrent, identical requests into a single execution instead of repeating the same work for each caller.

**Why it exists:** Establishing a new database connection pays for a TCP handshake, TLS negotiation, authentication, and — for PostgreSQL specifically, which allocates a dedicated OS process per connection — real process-creation overhead on the server side. Doing that per request wastes far more capacity than the request itself consumes. Separately, when many concurrent callers ask for the same thing at once (a cold cache key, a popular read), computing or fetching it once and sharing the result avoids redundant load on whatever backs it.

**Options:**
- **Per-request connections** — open a new connection per request, close it when done.
- **A pre-allocated connection pool** — maintain a fixed set of live, authenticated connections that requests lease and return.
- **Request coalescing** — detect concurrent, duplicate in-flight requests and serve all callers from a single execution.

**Trade-offs:** Per-request connections never block a caller waiting for a shared resource, but pay full setup cost on every request and can overwhelm a downstream server under load, since each connection consumes real server-side resources. A connection pool amortizes setup cost to near zero for the common case, at the cost of a hard concurrency ceiling: once the pool is exhausted, additional requests queue for a slot rather than fail fast, inflating tail latency under a spike. Request coalescing eliminates duplicate downstream load entirely for the collapsed requests, at the cost of coordination logic and a small added latency for whichever caller triggers the shared execution and waits for it to complete on behalf of the rest.

**When to choose each:** [Strong Recommendation] Connection pooling by default for any service maintaining persistent, high-volume communication with a database or backend — per-request connections are defensible only for infrequent, short-lived processes like a CLI script or a rarely-invoked serverless function. Coalescing wherever many callers plausibly request the same data at once — a hot cache key, a popular read path.

**Common failure modes:** *Thundering-herd connection exhaustion.* A stateless service auto-scales to absorb a load spike; each new instance opens its own local connection pool, and the fleet collectively attempts far more simultaneous connections than the database can accept, exhausting its connection limit and taking down the service the scaling was meant to protect. This is the same failure Ch 08 already named at the system level, recurring here at the specific resource of a database connection slot. Because PostgreSQL's per-connection process model makes it especially sensitive to this, a connection proxy such as PgBouncer is commonly placed in front of the cluster to absorb connection churn the application tier would otherwise pass straight through.

**Example:** Database-backed application servers universally pool connections because repeated TCP and database session setup becomes the bottleneck long before query execution itself does.

### Tail Latency Is a Different Problem Than Average Latency

Every lever above trades some latency for throughput — which makes it essential to know *which* latency is being traded away. A system's average or median latency (p50) can look excellent while a meaningful fraction of requests, its tail (commonly reported as p95, p99, or p99.9), take far longer. This is not a more extreme version of the average; it is frequently caused by a different mechanism entirely — queueing behind a batch, contention for a pooled resource, or a cold cache — and users experience individual requests, not a statistical average. Ch 73 already established the percentile SLI as the correct way to express this; a service should be evaluated and paged against a latency percentile, not a mean, because an excellent average and a rotting tail are fully compatible states for the same system to be in.

The stakes are highest in synchronous fan-out architectures: if a single user action triggers dozens of parallel downstream calls, the end-to-end request's latency is bounded by the *slowest* of those calls, not their average — so even a small per-service tail-latency probability compounds quickly into a much larger probability that the overall request hits some service's tail.

### Why Smart Engineers Disagree

The sharpest disputes over these levers are not about whether batching improves throughput — everyone agrees it does. They are about how much tail latency a system should tolerate in exchange for it.

Engineers building high-throughput, asynchronous pipelines — stream processors, large batch systems, systems like Kafka — treat median throughput as the metric that matters and tail latency as an acceptable cost. They configure large batch sizes, deep queues, and relaxed flush intervals deliberately, on the reasoning that maximizing total work completed per unit of hardware spend is the correct objective, and that protecting the slowest fraction of a percent of requests is a tax the workload doesn't need to pay.

Engineers operating synchronous, user-facing services take the opposite position, precisely because of the fan-out amplification described above: a single slow downstream call inside a 50-way parallel fan-out can turn a normally-fast median request into a slow one, so they strip out deep queues, enforce aggressive timeouts, and treat any batching delay on the request-serving path as a defect rather than an efficiency.

Both positions are correct for the topology they're describing. The resolution tracks where a component sits, not which side of the debate is "right": batch aggressively in decoupled, asynchronous background paths where nothing downstream is waiting synchronously, and suppress batching-induced latency deliberately at any synchronous, user-facing edge where a slow tail on one call becomes a slow response for the whole request.

---

Ch 88 takes up the lever this chapter deliberately did not cover: caching, as the specific mechanism for reducing latency by avoiding repeated work entirely rather than amortizing its cost.
