# Ch 69 — Logging: What to Log and at What Level

**Prerequisites:** [Complexity Is the Enemy](../part01-systems-thinking/ch02-complexity-is-the-enemy.md) (accidental complexity), Principle 6, [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (MTTR), [Error Handling Contracts](../part03-api-design/ch21-error-handling-contracts.md) (correlation ID), [What to Document vs. What to Leave to the Code](../part08-documentation/ch64-what-to-document-vs-what-to-leave-to-the-code.md) (coverage vs. usefulness)

**New vocabulary introduced:** actionability test

**Key takeaways:**
- A log line belongs in the system only if it passes the actionability test: a specific person, at some later point, makes a different decision because it exists. "Was this interesting to write" is not the test; "would this change what I conclude" is.
- [Strong Recommendation] Log levels classify who is expected to act and how urgently, not how the author felt while writing the code. The identical event can legitimately be DEBUG in one service and ERROR in another, depending on who is expected to respond and when.
- [Consensus] Structured, machine-parseable fields beat free-text messages once logs are aggregated across more than a handful of services. Every request-scoped entry should carry the correlation ID from Ch 21 so a request's full history is one indexed query instead of a manual reconstruction.
- Logging is a cost center, not a free byproduct of execution: every retained line is paid for in ingestion, storage, index size, and query latency — paid at exactly the moment, an active incident, when that latency is least affordable. Logging on the chance it's needed repeats Ch 64's documentation-coverage mistake with a runtime signal instead of a written one.
- A log is a second, less-audited copy of application data. What gets logged, and what parses it, are both part of the system's attack surface, not a debugging convenience — flagged here, owned by Part XI, Ch 83.

---

Every line an application logs in production is generated, serialized, transported, indexed, and stored — a runtime instance of the same information-cost argument Ch 02 makes about complexity and Ch 30 and Ch 64 make about comments and documentation. An unread log line that restates a step already visible in the code is no more valuable than a comment that restates the code next to it; it is pure accidental complexity injected into the observability pipeline, paid for whether or not anyone ever queries it. This chapter covers what belongs in a log once logging is the right tool. Whether a log, a metric, or a trace is the right tool for a given question at all is Ch 70's subject.

### Decision: Apply the Actionability Test, Not a Severity Instinct

**What it is:** The filter for whether an event belongs in a log at all: would a specific person make a different decision because this line exists, not whether the event felt significant to the engineer who wrote it.

**Why it exists:** Developers default to asking "is this interesting?" — a code path was reached, a branch was taken, a helper function ran to completion. Operators asking "would this change what I conclude?" get a much smaller answer. Every line that passes the developer's test but fails the operator's test is noise that someone else pays to store and someone else has to scroll past during an incident.

**Options:**
- **Severity-driven logging** — emit a line whenever an execution state feels notable to the engineer writing the code (a database row inserted, a function entered, a branch taken).
- **Action-driven logging** — restrict lines to state transitions, invariant violations, boundary crossings, and anomalies that require a specific downstream response or investigation.

**Trade-offs:**

| | Pros | Cons |
|---|---|---|
| Severity-driven | Minimal upfront design; produces a detailed local trace during initial development | Massive noise at production scale; high log-to-signal ratio buries the failures that actually matter |
| Action-driven | Small production footprint; every line returned by an incident query carries diagnostic weight | Requires deliberate judgment at write time and in review; a genuinely novel failure mode can fall outside what anyone anticipated logging |

**When to choose each:** [Strong Recommendation] Action-driven logging for every production service. Severity-driven logging is acceptable only in ephemeral local branches or short-lived prototypes, never past that point.

**Common failure modes:** *The Warning Flood.* A service logs a WARN every time a user submits an invalid form field or an expired token — both routine outcomes of normal traffic and automated scanners. Because these fire constantly, the operational team stops reading WARN-level output entirely. When a real anomaly — an intermittent database replica connection drop — starts logging at the same level, it produces the same reflexive dismissal as the thousands of routine warnings before it, and the team misses it until the outage is already user-visible.

**Example:** A payment service should log `payment_authorized`, `authorization_declined`, `gateway_timeout`, and `fraud_rule_triggered` — each a state transition someone might need to act on. It should not log that its internal `validateCard()` helper ran; a future reader needs to know that validation failed and which rule failed, not that a function executed.

---

### Decision: Log Levels Are a Triage Classification, Not a Record of Developer Sentiment

**What it is:** A log level tells whoever is filtering output how to treat the event during investigation — it does not measure how surprised or frustrated the engineer was when they wrote the line.

**Why it exists:** Without a shared, disciplined meaning attached to each level, filtering breaks down entirely. The convention traces back to syslog's long-standing severity taxonomy, which established that operational events should be classified by significance to a responder, not narrated chronologically. An ERROR should mean "this operation failed and needs investigation," not "this surprised me while I was writing the code."

**Options:** At minimum: `DEBUG`, `INFO`, `WARN`, `ERROR`. Some systems add `TRACE`, `FATAL`, or `CRITICAL` as finer gradations of the same idea.

**Trade-offs:**

| Level | What it should mean | Common misuse |
|---|---|---|
| DEBUG | Rich diagnostic detail, filtered out entirely in standard production operation | Left enabled in production, where it dominates ingestion volume for no corresponding benefit |
| INFO | A macro-level lifecycle event — service startup, config reload, an expected outcome like a declined card | Used for every internal step, becoming a second DEBUG stream |
| WARN | A non-fatal anomaly worth automated tracking but no immediate response | Used for routine, expected user error (see the Warning Flood, above) |
| ERROR | A localized failure requiring investigation or intervention | Used for expected, already-handled outcomes, which erodes the level's meaning until nobody trusts it |

**When to choose each:** The same underlying event can be legitimately classified differently in different services, because different operators are expected to respond differently. A failed login from an incorrect password is ordinarily INFO — the authentication system behaved exactly as designed. A failure to reach the authentication database is ERROR — the service failed to perform its function. An expected, automatically retried network timeout may be WARN in one service and DEBUG in another, if the retry completely masks the failure from any caller.

**Common failure modes:** Severity inflation — expected, already-handled outcomes (a declined card, an invalid token) get logged as ERROR because they *feel* like failures. ERROR volume grows in lockstep with normal traffic, and the one ERROR entry that represents an actual infrastructure failure is indistinguishable from the thousands of routine ones around it. What happens once that erosion feeds an alerting pipeline is Ch 71's subject, not this chapter's — but the erosion itself starts here, at the point a level is assigned.

**Example:** The classic Unix syslog severity scale (`LOG_EMERG` down to `LOG_DEBUG`) is the industry's oldest working example of this discipline: levels map to operational posture, not authorial emotion, and infrastructure at the aggregation boundary is configured to drop `DEBUG` entirely under normal operation.

---

### Decision: Structured Fields Beat Free-Text Messages at Any Real Scale

**What it is:** The choice between emitting an unconstrained, human-readable string per log line versus a stable set of machine-parseable key-value fields, typically serialized as JSON.

**Why it exists:** A free-text line (`"User 98321 connected from 192.168.1.50"`) reads well for one engineer tailing a single file locally. It is expensive for a centralized log platform processing thousands of events per second across hundreds of services — extracting a specific field requires a regular expression that a routine text change can silently break, at which point the downstream query or dashboard built on it goes quietly blind.

**Options:**
```
Free-text:
[INFO] 2026-07-02 09:15:00 - User 98321 connected from IP 192.168.1.50

Structured:
{
  "timestamp": "2026-07-02T09:15:00Z",
  "level": "INFO",
  "event": "user_connected",
  "user_id": "98321",
  "client_ip": "192.168.1.50",
  "correlation_id": "req-01j1p5x7"
}
```

**Trade-offs:** Free-text costs nothing to write and nothing at runtime to serialize, but is effectively unqueryable at scale — filtering on a specific field means a full-text scan or a fragile regex extraction, either of which saturates the log cluster's CPU during exactly the high-volume moment a query is most needed. Structured fields turn logs into a queryable dataset: a centralized backend such as an ELK-style pipeline can index a specific key directly and filter billions of records in milliseconds, at the cost of a small, continuous serialization tax on the application's execution path and a real discipline requirement around field naming.

**When to choose each:** [Consensus] Structured JSON for any production service feeding centralized aggregation — which, past a handful of services, is effectively all of them. Free-text remains acceptable for a standalone CLI tool or local script whose entire audience is one engineer watching a terminal.

Every request-scoped structured entry should include the correlation ID introduced in Ch 21. Even before Ch 72's full distributed trace context exists, a consistent `correlation_id` field lets an on-call engineer run one indexed query and reassemble a request's chronological history across thread and service boundaries — a floor of visibility a service gets almost for free, well before it invests in tracing infrastructure.

**Common failure modes:** *The schema collision.* Team A logs a field named `context` as a plain string (`"context": "database_retry"`). Team B logs a field with the same name as a nested object (`"context": {"error_code": 500, "retry_count": 3}`). When the centralized indexer encounters both shapes for the same key, it rejects the conflicting payloads — Team B's failure telemetry silently disappears from the index during a live incident, and nobody notices until they go looking for it and find nothing there. A narrower version of the same failure: a service generates a correlation ID at request start but only logs it in the first line, so every subsequent entry for that request is unlinked from it and the timeline can no longer be reassembled by that field alone.

**Example:** Structured JSON logs are the input format centralized platforms such as Elasticsearch/ELK are built to index directly, without the extraction step a free-text pipeline requires.

---

### Decision: Treat Ingestion Volume as a Budget, Not a Convenience

**What it is:** Whether to log broadly on the chance that some future, unpredicted failure needs the context, versus deliberately restricting standard-operation logging and capturing deep detail only once an actual error occurs.

**Why it exists:** "Log everything, in case it's needed" is Ch 64's documentation-coverage mistake wearing a different costume: volume that satisfies an instinct about future usefulness rather than a demonstrated one, at real and continuous cost. That cost is not hypothetical — ingestion, indexing, and query latency scale with volume, and at high enough throughput a logging platform's bill can rival the cost of the primary compute infrastructure it's observing. Verbose, unreviewed logging also creates a second data-exposure surface: incidental secrets or personal data logged as a side effect become a copy of sensitive data sitting in a less-audited store, a boundary this chapter flags but leaves to Ch 83.

**Options:**
- **Comprehensive, on-the-chance ingestion** — log broadly across execution paths and payload contexts, relying on storage capacity to retain it for whatever investigation eventually needs it.
- **Intentional filtering** — restrict standard-operation logging to boundary state transitions, and rely on dynamic log-level changes or explicit error-context capture to go deep only once something has actually gone wrong.

**Trade-offs:** Comprehensive ingestion means the forensic evidence for a rare, non-reproducible failure is already captured when it happens — at the cost of a large, continuously growing financial and operational bill, degraded query performance across the whole platform, and an expanded exposure surface for anything sensitive that ends up in the payload. Intentional filtering keeps ingestion volume and cost predictable and keeps queries fast even under load, at the cost of needing more deliberate tooling — dynamic log-level control, or a local buffer flushed only on error — to recover deep detail when a genuinely novel failure does occur.

**When to choose each:** [Strong Recommendation] Intentional filtering as the default for any mature, high-traffic, multi-tenant, or regulated system, where cost predictability and data-exposure discipline are hard constraints. Comprehensive ingestion is defensible only in early-stage, low-volume, or short-lived systems where compute cost is trivial relative to the value of not having to guess.

**Common failure modes:** A service under an on-the-chance logging policy logs the full request body of every incoming request, including malformed ones. An unthrottled attack sends malformed payloads at volume; ingestion spikes, saturating the log backend's disk I/O; the log cluster falls over under the load, and the team investigating the attack loses visibility into every other service at the exact moment they need it most — the logging policy meant to provide forensic insurance during an incident instead causes the outage that erases it.

**Example:** The Log4Shell vulnerability (CVE-2021-44228) demonstrated that a logging library is part of a system's attack surface, not a passive utility sitting outside it. Applications that logged raw, unvalidated user input — an HTTP `User-Agent` header, for instance — through a vulnerable version of Apache Log4j allowed a crafted string to trigger remote code execution inside the logging call itself. Logging everything without validating or scrubbing what gets logged is not a neutral default; it's an architectural risk decision, made by omission.

---

What triggers a human to be paged over something a log recorded is Ch 71's decision, not this chapter's. Correlating a correlation ID across service boundaries into a full causal trace — trace ID, span ID, propagation — is Ch 72's mechanism; this chapter only establishes that the field belongs in every entry. And the mechanics of keeping a secret out of a log line in the first place belong to Ch 83, not here.

### Why Smart Engineers Disagree

The disagreement is not about whether logging is useful — it is about how much production visibility is worth its ongoing cost, and it splits engineers who have been burned by missing information from engineers who have been burned by the bill and the noise.

One position holds that disabling deep diagnostic logging in production is a false economy: distributed systems fail in emergent, non-deterministic ways no staging environment reproduces, and without step-by-step state the responder is left guessing, directly extending MTTR. This position treats the resulting storage and ingestion cost as a justifiable price for engineering clarity, and pushes for DEBUG-level detail to remain available, if not always active, at all times.

The opposing position treats continuous, high-volume production debugging as a design failure in its own right. At meaningful throughput, the string allocation, serialization, and I/O cost of emitting deep-detail logs on every request measurably throttles the service producing them, and the resulting ingestion bill grows with traffic regardless of whether anything is actually wrong. This position argues that needing a continuous text narrative to confirm a service is behaving correctly is itself a symptom of insufficient modular design (Part IV) — not a logging gap.

Both positions are correct for the system each one has in mind. A low-volume, high-consequence orchestration path — a multi-step billing settlement — can absorb deep logging on every request; the cost is negligible against the cost of an undetected miscalculation. A high-throughput ingestion path — a message router processing tens of thousands of events per second — cannot; a log line per event there throttles the very system it's meant to observe. The resolution most mature systems converge on is decoupling recording from extraction rather than picking one philosophy globally: keep production-filtered, action-driven logging as the default everywhere, but hold a local, in-memory ring buffer of recent DEBUG-level detail per request. On success, that buffer is overwritten by the next request's detail. Only when an actual error is detected does the service flush it to the central log store — paying the full ingestion cost of deep detail exactly once, for the one request that needed it, and nothing for the millions around it that didn't.
