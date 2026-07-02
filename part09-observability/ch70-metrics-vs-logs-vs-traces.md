# Ch 70 — Metrics vs. Logs vs. Traces

**Prerequisites:** [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md) (optimization targets), [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (MTTR, partial failure), [Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md) (Little's Law), [Coupling and Cohesion at the Architecture Level](../part02-software-architecture/ch13-coupling-cohesion-architecture-level.md) (service boundaries), [Sync vs. Async Communication](../part02-software-architecture/ch17-sync-vs-async-communication.md) (Kafka), [Error Handling Contracts](../part03-api-design/ch21-error-handling-contracts.md) (correlation ID), [Logging: What to Log and at What Level](ch69-logging-what-to-log-and-at-what-level.md) (actionability test)

**New vocabulary introduced:** cardinality (metric)

**Key takeaways:**
- [Consensus] Logs, metrics, and traces are not interchangeable formats for the same data — each has a distinct structural shape (discrete event, aggregable time series, causal graph) built to answer a different class of question at a different cost. None replaces the other two.
- Identify the question before picking the signal: "is the system healthy right now" routes to a metric; "why did this specific request fail" routes to a trace, or a log found via its correlation ID (Ch 21); "what happened around this event three hours ago" routes to a log.
- [Consensus] Cardinality — the number of distinct label-value combinations a metric can take on — is why metrics stay cheap. A label restricted to a small, bounded set of values aggregates efficiently across millions of events; a label containing a near-unique value per event (a user ID, a request ID) collapses that advantage and makes the metric cost, and behave, like a log.
- Kafka consumer lag is the resolved example of a metric-shaped operational question: whether a consumer is keeping pace with a topic is answered by one low-cardinality number, not by logging every message that passes through.
- The signal choice is downstream of Ch 69's actionability test, not a replacement for it — a trace or metric that nobody acts on is exactly as much waste as a log line nobody reads.

---

Ch 69 established what belongs in a log once logging is the right tool. This chapter answers the question one level up: which of the three observability signal types is the right tool at all, for a given operational question, at what cost. The three already exist by this point — logs (Ch 69), and metrics and traces, both introduced here. Traces are named and placed in this taxonomy; the mechanics of context propagation and sampling that make tracing work are Ch 72's subject, not this chapter's.

### Decision: Match the Signal's Shape to the Question, Not the Other Way Around

**What it is:** Logs, metrics, and traces are not three formats for the same underlying data — each intentionally sacrifices a different kind of detail to make a different kind of query cheap.

**Why it exists:** A common assumption is that any of the three could, with enough effort, answer any observability question. In principle that's true; in practice each signal's cost structure makes it viable for some questions and prohibitive for others. Using the wrong signal for a question means paying more to learn less.

**Options:**

| Signal | Structural shape | Answers | Cost profile |
|---|---|---|---|
| **Log** | A discrete, timestamped event with arbitrary structure | "What happened, specifically, at this moment?" | Scales linearly with event volume — cheap per query, expensive in aggregate |
| **Metric** | A numeric, pre-aggregated time series, deliberately low-cardinality | "How much? How often? Is the system healthy right now?" | Near-flat, largely decoupled from traffic volume |
| **Trace** | A causal graph of spans following one request across service boundaries | "Where did the time go, for this one request?" | Expensive per request; requires sampling at real scale (Ch 72) |

**Trade-offs:** A log preserves full per-event detail but becomes expensive and slow to search once volume grows past what a targeted query can narrow down quickly. A metric answers aggregate questions almost instantly regardless of scale, precisely because it has already thrown away individual request detail before storing anything. A trace is the only one of the three that reconstructs causality across a service boundary, but capturing it for every request is often prohibitively expensive at meaningful throughput.

**When to choose each:** Metrics for continuous system-health monitoring, dashboards, and automated scaling. Traces for isolating where time or failure occurred within one specific, already-identified request. Logs for reconstructing what happened around a specific historical event.

**Common failure modes:** *The log-derived metric trap.* An organization without dedicated metrics infrastructure configures a regex scraper to derive throughput and error-rate numbers from its text log stream. During a downstream database lockup, log volume spikes; the scraper falls behind in near-real time, and the alerting built on top of it goes silent at the exact moment a system-wide failure needs it most — because the "metric" was never actually decoupled from log ingestion cost in the first place.

**Example:** An API starts returning elevated latency. A metric dashboard shows the aggregate latency curve rising — confirming *that* something is wrong. A trace on one of the slow requests shows most of the time was spent waiting on a downstream service call — showing *where*. A log search around that timestamp turns up a configuration change to that downstream service moments earlier — showing *why*. No single signal answers all three questions; together they do.

---

### Decision: Start From the Question, Not From Whichever Signal Is Already Deployed

**What it is:** The concrete selection test for choosing a signal — identify what's actually being asked before reaching for a tool.

**Why it exists:** Teams tend to standardize on whichever signal their existing tooling makes easiest, then bend every question to fit it — an organization with mature log search tries to answer aggregate health questions through log queries; an organization invested heavily in metrics tries to cram per-request detail into metric labels (the specific failure the next section names). Both directions produce the same result: more cost, less answer.

**Options:**

| Operational question | Signal | Why |
|---|---|---|
| Is the service healthy right now? | Metric | Aggregated system state |
| Is traffic or latency trending up? | Metric | Time-series trend |
| Why did this specific customer's request fail? | Trace, or log via correlation ID | Individual execution history |
| Where did this one request spend its time? | Trace | Causal, cross-service latency attribution |
| What happened three hours ago, before the outage? | Log | Discrete historical event |
| Which configuration changed right before the failure? | Log | Discrete historical event |

**When to choose each:** Ask first whether the investigation concerns the system as a whole, a specific past event, or one specific request — the answer to that question determines the signal, not the other way around.

**Common failure modes:** An organization tries to answer "why did this customer's checkout fail" purely from a metrics dashboard. The dashboard can show that the checkout error rate rose at a given minute; it structurally cannot show which customer, which input, or which downstream call caused any individual failure — that information was discarded at aggregation time, by design, and no amount of dashboard tooling gets it back.

**Example:** A customer reports a checkout request that took twelve seconds. A distributed trace on that request shows nine of those seconds were spent waiting on an inventory service call. A metric dashboard, showing only aggregate checkout latency, cannot localize the twelve seconds to any specific dependency — it wasn't built to.

---

### Decision: Keep Metric Labels Low-Cardinality — Never Route a Unique Value Through One

**What it is:** Cardinality is the number of distinct label-value combinations a metric produces. A metric stays cheap only because that number is kept small and bounded.

**Why it exists:** A metrics engine achieves its speed by maintaining one indexed time series per unique combination of metric name and label values. A label restricted to a handful of values (an HTTP method, a status code, a region) produces a small, stable number of series regardless of traffic volume. A label containing a near-unique value per event — a user ID, a request ID — produces close to one new time series per event, which is a log's cost structure wearing a metric's name.

**Options:**
- **Low-cardinality labels** — restrict metric dimensions to small, predictable enumerations (`http_method`, `status_code`, `region`).
- **High-cardinality labels** — attach a per-request or per-user identifier directly to a metric's label set for fine-grained dashboard filtering.

**Trade-offs:** Low-cardinality labels guarantee flat, predictable memory and storage cost regardless of traffic, and keep dashboards fast even under load — at the cost of not being able to filter a metric down to one individual account or request; that granularity has to come from a log or trace instead. High-cardinality labels let an engineer isolate one user's behavior directly on a graph, but the index size grows with the number of unique values seen, not with the number of metrics defined — at real scale, this is not a performance degradation, it's a collapse of the metrics store itself.

**When to choose each:** [Consensus] Low-cardinality labels only, for every metric, without exception. Never attach an unbounded identifier to a metric label; if a value needs to be tracked per-request or per-user, that value belongs in a log or a trace attribute, both of which are built for high-cardinality, sparse indexing.

**Common failure modes:** A developer adds a `user_id` label to a request-duration histogram to make per-customer filtering possible on a dashboard. In staging, with ten test accounts, nothing looks wrong. In production, with millions of distinct customers, the metrics engine attempts to allocate and index a separate time series per user ID it has ever seen, exhausts host memory, and enters an out-of-memory crash loop — taking down monitoring visibility for every other service at the same time, often while masking an unrelated, concurrent outage the team can no longer see.

**Example:** Prometheus's data model makes this discipline explicit: a series is keyed by its full label set (`http_requests_total{service="checkout", status="500", method="POST"}`), and teams that maintain cardinality discipline typically enforce it mechanically — a PromQL linter in CI blocking any instrumentation change that introduces an unbounded label — rather than relying on code review alone to catch it (Principle 8).

---

### Decision: Observe an Event-Driven Pipeline's Backlog With a Metric, Not a Log Line Per Message

**What it is:** Whether a message queue's processing health is tracked through a low-cardinality lag metric or reconstructed by parsing per-message processing logs.

**Why it exists:** Ch 17 deferred Kafka's operational visibility to this Part. In a decoupled, event-driven pipeline, per-message logging at the volume a broker like Kafka operates at is prohibitively expensive, and it answers the wrong question anyway: whether a consumer is keeping pace with its topic is an aggregate question about backlog, not a question about the contents of any individual message.

**Options:**
- **Consumer lag as a metric** — expose the numeric distance between the latest produced offset and the current consumer group's committed offset as a time series.
- **In-line per-message logs** — emit a log entry for every message polled, processed, or acknowledged.

**Trade-offs:** A lag metric has near-zero storage cost and gives an unambiguous, real-time backlog signal that plugs directly into an autoscaling rule — at the cost of saying nothing about what's inside any specific message. Per-message logs preserve a full, auditable transaction record — at the cost of I/O and storage overhead that scales with message volume, and a real risk that a retrying consumer logs the same failing message repeatedly, multiplying the cost without adding new information.

**When to choose each:** [Strong Recommendation] Consumer lag as the default, continuous signal for pipeline health and autoscaling. Reserve per-message logging for capturing why a specific message failed — a dead-letter transition or a structural error — and keep ordinary successful processing at DEBUG or below, per Ch 69's actionability test.

**Common failure modes:** *The poison-pill starvation outage.* A malformed message reaches a Kafka topic. The consumer pulls it, throws during parsing, and crashes; the framework restarts the consumer at the same offset, which reprocesses the identical message and crashes again, indefinitely. Because the team monitors health through log volume rather than lag, the continuous stream of crash-and-retry log lines looks like activity — the service appears busy, not stuck. A lag metric, had it been tracked, would show a vertical, sustained climb on that partition; without it, the team stays blind to the pipeline's actual halt for hours.

**Example:** Kafka manages consumer group offsets internally; an exporter such as `kafka-lag-exporter` or the Prometheus JMX Exporter surfaces `kafka_consumergroup_lag` as a clean, low-cardinality series labeled only by topic and consumer group. An alert threshold on that single number can page an engineer, or trigger Kubernetes autoscaling (via KEDA) for the consumer deployment, before a growing backlog turns into a downstream data-consistency or SLA problem.

---

Which of these three signals is worth waking a human for is Ch 71's decision, not this chapter's. The trace data model, context propagation across a call chain, and sampling strategy at scale are Ch 72's subject — this chapter only establishes that a trace exists and what question it answers relative to the other two. And the service boundaries a trace crosses are Part II's design, not this Part's to redraw; observability observes them, it doesn't reshape them.

### Why Smart Engineers Disagree

The sharpest disagreement in this taxonomy isn't between logs and metrics — it's about how far to push distributed tracing adoption, and it splits engineers by how much cross-service failure they've had to debug without it.

One position treats isolated, per-service logging as an outdated pattern for a distributed system: every request path should carry a full trace context from day one, with logs demoted to structured events attached to a span rather than standing alone. Reconstructing a distributed request by hand from scattered log lines and a correlation ID, in this view, is error-prone and slow exactly when speed matters most — and the data volume that comes with tracing everything is managed with aggressive, tail-based sampling that evaluates a request's outcome before deciding whether to keep it.

The opposing position treats blanket distributed tracing as an expensive, heavy dependency for what it actually returns in most systems: propagating trace context correctly across every service, thread pool, and async boundary takes real, sustained engineering discipline, and the infrastructure to store and query traces at scale is not cheap. This position argues that a disciplined structured-logging practice with a consistently propagated correlation ID (Ch 69, Ch 21) recovers most of tracing's practical diagnostic value at a fraction of the cost, and reserves full tracing for a thin, head-sampled slice of traffic rather than the default path.

Both are right for the architecture each has in mind. In a large organization with deeply nested microservice dependencies owned by dozens of independent teams, reconstructing a cascading failure without an automated causal graph becomes close to impossible, and the cost of full tracing infrastructure is justified by that alone. In a monolith or a shallow service topology — under roughly ten independently deployed boundaries — the same tracing infrastructure is accidental complexity with no problem left for it to solve. The practical resolution most systems converge on is not choosing a side permanently but keeping the transition cheap: enforce a single correlation ID through every protocol boundary from the start, regardless of scale, so that a system can move from correlation-ID-linked logs to full distributed tracing exactly when its call-depth and organizational complexity demand it — not before, and not too late to matter.
