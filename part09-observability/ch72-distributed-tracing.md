# Ch 72 — Distributed Tracing

**Prerequisites:** [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (partial failure), [Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md) (Little's Law), [Coupling and Cohesion at the Architecture Level](../part02-software-architecture/ch13-coupling-cohesion-architecture-level.md) (service boundaries), [Error Handling Contracts](../part03-api-design/ch21-error-handling-contracts.md) (correlation ID), [Metrics vs. Logs vs. Traces](ch70-metrics-vs-logs-vs-traces.md)

**New vocabulary introduced:** trace context

**Key takeaways:**
- A trace context — trace ID, span ID, parent span ID — is the correlation ID from Ch 21 extended into a structure that captures causality, not only relatedness. Log correlation can tell you two events shared a request; only an explicit, propagated context can tell you which service called which, and how long the gap between them was.
- [Strong Recommendation] Trace propagation is a cross-team discipline, not a per-service implementation detail: one service that fails to forward its inbound context doesn't only lose its own visibility, it orphans every span downstream of it, breaking the trace for services that did everything correctly.
- Tracing every request is prohibitively expensive at real scale — sampling is a required design decision, not an optional refinement. Head-based sampling is cheap but structurally blind to rare failures; tail-based sampling catches them by evaluating a request before deciding whether to keep it, at the cost of buffering every in-flight request until it finishes.
- A trace is the direct observational tool for Ch 08's warning that local optimization can damage the global system: it is the one signal that can show a request's time was spent waiting between services, not inside any one of them — a category of cost no local profiler can see.
- This chapter goes deep on tracing mechanics; where tracing sits relative to logs and metrics is already settled by Ch 70, and alerting on trace-derived latency is Ch 71's decision, not this chapter's.

---

Ch 70 already placed tracing in the observability taxonomy: it's the signal that answers "where did the time go for this one request," at a cost the other two signals don't pay. This chapter goes deep on how that signal is actually produced — the data model, the propagation discipline it depends on, and the sampling trade-off that makes it affordable.

### Decision: Extend the Correlation ID Into an Explicit, Propagated Trace Context

**What it is:** Whether to reconstruct a request's cross-service history by searching logs for a shared correlation ID after the fact, or by having every service explicitly read, extend, and forward a structured trace context — trace ID, span ID, parent span ID — on every call it makes.

**Why it exists:** A correlation ID (Ch 21, extended into structured logs in Ch 69) lets an engineer find every log line belonging to one request. It cannot establish exact causality or timing between them. If a service calls three downstream dependencies in parallel, log correlation can't determine which call spawned which sub-request, or how much of the elapsed time was local processing versus network transit. A trace is built from spans — one span per unit of work, such as handling an HTTP request or executing a database query — connected by explicit parent-child relationships that form a causal tree, not only a chronologically sortable set.

```
Log correlation (relatedness only):
[Service A log] ── correlation_id: XYZ ── [Service B log]
(no parent-child structure; can't isolate network transit time)

Trace context (explicit causality):
[Service A — Span 101, Trace XYZ]
     │  wire header: TraceID=XYZ, ParentSpanID=101
     ▼
[Service B — Span 102, Trace XYZ, Parent=101]
```

**Trade-offs:** Relying on log correlation alone requires no tracing middleware and adds no wire-protocol dependency, but it's structurally incapable of mapping concurrent or asynchronous execution, or isolating network-transit latency from local processing time. An explicit trace context delivers a precise causal graph — the exact delay between an upstream dispatch and a downstream receive — at the cost of requiring every participating service to correctly extract and forward the context on every call, with no exceptions.

**When to choose each:** [Strong Recommendation] Explicit trace context propagation for any system with real multi-service decomposition or asynchronous messaging, where isolating cross-service latency materially affects MTTR. Log correlation alone remains adequate for a monolith or a shallow, synchronous call chain of a few hops, where the topology is static and small enough to hold in one's head.

**Common failure modes:** *The context breakage void.* A platform runs distributed tracing across twenty services. One team refactors an internal proxy from an instrumented HTTP client to a raw socket client to shave serialization overhead, and the new client silently drops the inbound trace headers instead of forwarding them. Business traffic through the proxy is unaffected — but every trace passing through it now terminates there in the visualization tool, and the nineteen services downstream generate detached "orphan root" traces with no parent. The platform hasn't lost tracing for one service; it's lost the ability to see the causal chain for every request that passes through that one broken link.

**Example:** OpenTelemetry implements this as the W3C Trace Context standard. An outbound HTTP call carries a header such as `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01` — a version byte, a globally unique trace ID, the parent span ID, and sampling flags. Every downstream service reads that header, mints its own child span ID, and forwards the updated context on whatever it calls next.

---

### Decision: Sample at the Head, or Buffer and Decide at the Tail

**What it is:** Whether to commit to keeping or discarding a trace at the moment a request enters the system (head-based sampling), or to buffer every span in memory until the request completes and decide only then, based on its outcome (tail-based sampling).

**Why it exists:** Recording, transporting, and indexing a trace for every request at real production throughput is prohibitively expensive — tracing overhead can rival the cost of the business logic it's observing. Sampling is not optional. But a fixed head-based rate drops data blindly: at a 1% sample rate, a rare failure affecting 0.05% of traffic is filtered out almost entirely, leaving no forensic evidence for exactly the incident an engineer needs to investigate.

**Options:**
- **Head-based, fixed-percentage sampling** — hash the incoming trace ID at the edge and commit to keeping or dropping it instantly, before the request executes.
- **Tail-based, buffered sampling** — let every span generate normally, route them to a collector that holds the full request's spans in memory until it finishes, then apply a rule (keep anything that errored or exceeded a latency threshold; discard the rest).

**Trade-offs:** Head-based sampling has near-zero memory overhead and requires no buffering infrastructure — a request that's dropped costs nothing downstream — but it is statistically blind to rare, high-value failures precisely because it decides before anything interesting has happened. Tail-based sampling guarantees high-fidelity capture of every error and every long-tail latency spike while discarding the ordinary, uninteresting traffic that provides no diagnostic value, at the cost of a collector infrastructure that must hold every in-flight request's full span tree in memory until it resolves.

**When to choose each:** [Strong Recommendation] Head-based sampling for ultra-high-throughput, latency-insensitive traffic where data volume is the dominant constraint and macro trends matter more than single-request forensics. Tail-based sampling for business-critical paths — checkout, financial transactions — where capturing the exact signature of every failure or slow outlier is worth the added infrastructure.

**Common failure modes:** *The collector memory avalanche.* A platform runs tail-based sampling to guarantee it never misses a failure. A downstream payment gateway degrades, and response times climb from 50ms to 30 seconds across the fleet. Because tail-based sampling must hold every span until its request resolves, and requests are now taking 30 seconds instead of 50 milliseconds to resolve, the collector's in-memory buffer balloons far past its capacity and crashes in an out-of-memory loop — blinding the team to all tracing data at the exact moment the incident it was built to catch is happening.

**Example:** Google's original Dapper paper, the foundational design nearly every modern tracing system descends from, documented that a stable, low, head-based sampling rate (on the order of 0.01%) was sufficient to observe a web-scale system's behavior without meaningfully taxing host compute budgets. Systems that need per-failure fidelity instead route every span to a local collector whose tail-sampling policy explicitly retains any trace carrying an error status or exceeding a duration threshold, flushing ordinary successful traffic without persisting it.

---

### Decision: Read the Trace's Gaps, Not a Local Profiler, for a Cross-Service Bottleneck

**What it is:** Whether to diagnose elevated request latency by attaching a local execution profiler to a single service's process, or by reading the empty space between spans in a distributed trace.

**Why it exists:** This is Ch 08's Little's Law warning made directly observable. When a request slows down, the instinct is to profile the service that returned it — thread states, heap allocation, query duration. But in a distributed system, a request's time is routinely dominated not by any service's own execution, but by the space *between* services: a socket buffer, an exhausted connection pool, a queue behind a rate limiter. A local profiler sees that time as passive "waiting" and has no way to attribute it to a structural cause; a trace shows it as an explicit, measurable gap between a parent span's dispatch and a child span's start.

**Trade-offs:** A distributed trace immediately exposes connection-pool starvation, queueing, and cross-service cascading delay — the exact class of problem Ch 08 warns a locally-optimized service can't see in its own metrics — but it has no line-of-code fidelity: a span can show a database call took 4.2 seconds without showing which query or allocation inside it was responsible. A local profiler gives exactly that line-level detail, but is structurally blind to network topology; it cannot distinguish "waiting on a slow downstream dependency" from "waiting on I/O for any other reason."

**When to choose each:** Reach for a distributed trace first whenever the investigation concerns latency inflation, timeout cascades, or throughput loss across a multi-service boundary. Reach for a local profiler (the performance-tuning toolkit belongs to Part XII) only after a trace has already isolated the delay to time genuinely spent executing inside one specific, uninterrupted host process — not waiting on anything external to it.

**Common failure modes:** *The local optimization blunder.* A checkout endpoint's p99 latency degrades by two seconds. An engineering team spends three weeks profiling locally, trimming memory allocations and object creation, and shaves 15 milliseconds off local CPU time — a rounding error against the two-second regression. A trace on the same endpoint would have shown the two seconds as one uninterrupted gap between a gateway's outbound call and the downstream service's inbound span: connection-pool exhaustion at the network perimeter, not application code. The team optimized a component that was never the bottleneck, the textbook case of Ch 08's warning that local optimization can leave — or worsen — the global problem entirely untouched.

**Example:** An API router forwards traffic to an order service. During a latency spike, the router's outbound span (`http.client.duration`) reads 1500ms, while the order service's inbound span (`http.server.duration`) for the same call reads only 50ms. The 1450ms gap between them is not attributable to either service's own code — it's visible directly in the trace as time spent in transit or queued at a connection boundary, pointing the responder at network or connection-pool configuration instead of application logic.

---

Where tracing sits relative to logs and metrics — which question each one answers — is already settled in Ch 70; this chapter only goes deep on tracing's own mechanics. Whether a trace-derived latency signal is worth paging a human over is Ch 71's decision. And propagating a trace context across an untrusted boundary carries its own exposure risk — it can leak internal topology to anyone able to read it — which this chapter flags but leaves entirely to Part XI, Ch 79.

### Why Smart Engineers Disagree

The disagreement isn't over whether tracing is worth having — it's over how the instrumentation that produces it should be written.

One position holds that relying on individual engineers to manually write span boundaries and propagate context by hand is a strategy that fails predictably: humans forget edge cases, and propagation hygiene erodes as a codebase changes hands over years. This position favors auto-instrumentation — runtime bytecode manipulation, compiler-level injection, or kernel-level probes (eBPF) — that hooks standard framework libraries (HTTP clients, gRPC stacks, database drivers) transparently, without requiring the application developer to write any tracing code at all. It prioritizes guaranteed, uniform baseline coverage over precision.

The opposing position treats auto-instrumentation as a source of accidental complexity in its own right: bytecode-rewriting agents and kernel probes intercept every internal framework call indiscriminately, including generic middleware loops that carry no business meaning, manufacturing large volumes of low-signal spans that inflate storage and dilute the trace with noise. This position holds that only the engineer who designed a service actually knows which boundaries represent a genuine unit of business work worth its own span, and argues for writing spans explicitly, by hand, using the tracing library's ordinary API.

The resolution tracks where in the stack the instrumentation sits. At the outermost infrastructure perimeter — standard HTTP or gRPC ingress, managed database clients — auto-instrumentation captures the structural, cross-service skeleton of a call graph with no ongoing developer effort, and the risk of missing a boundary there is low because the framework code changes rarely. Inside the core application layer, where the boundaries that matter are business decisions rather than framework calls, manual instrumentation is worth its cost: it's the only way to guarantee that a span actually represents a unit of work an engineer would recognize during an incident, tagged with the domain context — an order ID, a tenant, a decision outcome — that makes the trace actionable rather than merely complete.
