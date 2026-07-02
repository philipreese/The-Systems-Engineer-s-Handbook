# Part IX — Chapter Specifications

Pre-filled special instructions for each Part IX chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part IX: Part I (Ch 01's optimization-target framing applies directly —
observability exists to make an implicit optimization target measurable, and Principle 6 applies
to every signal collected exactly as it applies to any other process: an unread log line, an
unactioned alert, and an untracked metric are all overhead that was never worth its cost; Ch 07's
MTTR/MTBF paradigms and partial-failure argument are this Part's direct foundation, and Ch 07
itself forward-references Ch 71, Ch 72, and Ch 73 by number; Ch 08's Little's Law and
local-vs-global-optimization argument is Ch 72's direct setup, and Ch 08 forward-references Ch 72
by number); Part II (Ch 13's service-boundary and coupling arguments describe exactly what a
distributed trace crosses; Ch 17's sync-vs-async chapter forward-references this Part in the
context of Kafka's operational visibility); Part III (Ch 21's correlation ID is the mechanism Ch 69
and Ch 72 both extend — a single request identifier that ties a log line, an error response, and a
trace span back to the same call).

Several forward references from earlier parts must be honored here:
- Part I, Ch 07 deferred error budgets and SLOs (Ch 73), distributed tracing (Ch 72), and alerting
  strategy (Ch 71) to this Part by number.
- Part I, Ch 08 deferred distributed tracing as a system-level observation tool (Ch 72) to this
  Part by number.
- Part II, Ch 17 deferred Kafka's operational configuration and consumer visibility to this Part in
  general terms — resolved here in Ch 70 as a concrete example of a metric-shaped operational
  signal.
- Part VII, Ch 62 deferred the specific metrics and error budgets that gate a canary or promotion
  decision to this Part by number (Ch 73); that chapter owns the promotion mechanism, this Part
  owns the signal that gates it.
- Part VIII, Ch 68 deferred what triggers a responder to open a runbook (Ch 71) and what a runbook
  ultimately protects (Ch 73) to this Part by number, twice.

Boundary with adjacent Parts: Part VIII owns the runbook document a responder reads once paged;
this Part owns the decision to page them in the first place (Ch 71) and the target their response
ultimately protects (Ch 73) — chapters here reference the runbook as the destination an alert
points to, they don't re-cover its content. Part VII owns deployment and promotion mechanics —
canary traffic shifting, rollback execution; this Part owns only the metric that gates those
decisions (Ch 73), not the mechanism itself. Part II owns service boundary and coupling design;
this Part observes the boundaries Part II drew, it doesn't redesign them. Part XI owns security;
any chapter that brushes against credentials appearing in logs or traces should flag the boundary
to Ch 83, not treat secrets handling as its own topic. Part XII owns performance optimization and
profiling; where a metric or trace reveals a performance problem, this Part stops at observing and
measuring it — what to do about it belongs to Part XII.

This Part is small and progressively zooms out by design: Ch 69 decides what a single service
should record about its own behavior, Ch 70 places that recording alongside the other two signal
types and resolves which question each one actually answers, Ch 71 decides which of those signals
is worth waking a human for, Ch 72 goes deep on the one signal type that exists specifically
because no single service has the whole picture, and Ch 73 closes the Part by naming the target
all four of the preceding chapters ultimately serve.

---

## Ch 69 — Logging: What to Log and at What Level

```
This chapter opens Part IX by applying this handbook's information-cost argument to a runtime
signal instead of a written one: Ch 02's complexity argument and Principle 6 apply to a log line
exactly as they apply to a comment (Ch 30) or a page of documentation (Ch 64) — every line logged
is a line stored, indexed, and paid for whether or not anyone ever reads it, and an unread log line
that captures the wrong information is no more valuable than a comment that restates the code.

Cover the core test for what belongs in a log: not "how severe does this feel" but "who needs this,
and what decision does it let them make after the fact." Cover log levels (debug, info, warn, error
at minimum) as a triage mechanism for that test, not a measure of how urgently the author felt about
the event at write time — the same underlying event can be debug noise in one service and an error
in another depending on who's expected to act on it and when. Cover structured logging (stable
key-value or JSON fields) versus free-text messages as the chapter's central trade-off, taking a
position: structured logging is what makes a log queryable and machine-parseable at any scale
beyond a single engineer tailing a file, and every structured log entry should carry the correlation
ID introduced in Ch 21 so a single request's log lines can be reassembled across a service boundary
even before Ch 72's full trace context exists.

Cover the cost side explicitly and concretely: log volume is a real infrastructure cost (ingestion,
storage, query latency at read time) and a real security cost (secrets or personal data logged
incidentally become a second, less-audited copy of sensitive data) — flag the secrets boundary to
Ch 83 without expanding into it, the same way Ch 68 flagged it for runbooks. Cover the common
overcorrection directly: logging everything "just in case" produces the same failure Ch 64 already
named for documentation coverage — volume that satisfies an instinct rather than a genuine future
debugging need, at real ongoing cost.

Anchor in real systems: syslog's severity-level taxonomy as the long-standing industry baseline for
what a log level is supposed to mean; structured JSON logging as practiced by modern log pipelines
(the kind that feed tools like Elasticsearch/ELK or an equivalent aggregation backend) as the
concrete alternative to free-text logs; the Log4Shell (CVE-2021-44228) incident as a concrete,
widely known illustration that a logging library itself is part of a system's attack surface, not a
neutral utility — what gets logged, and what parses it, is a security-relevant design decision, not
an afterthought.

Do NOT cover: the taxonomy of when a log is the right signal versus a metric or a trace for a given
question (Ch 70 — this chapter only covers what belongs in a log once logging is the right tool);
alerting on log content (Ch 71); correlating logs across service boundaries into a full distributed
trace (Ch 72 — this chapter covers the correlation ID as a field carried in the log, not the
propagation mechanics across services); secrets management mechanics (Part XI, Ch 83).
```

---

## Ch 70 — Metrics vs. Logs vs. Traces

```
This chapter resolves the observability taxonomy the same way Ch 65 resolved the documentation
taxonomy: three signal types already exist by this point in the handbook — logs (Ch 69), metrics,
and traces (named here, expanded in Ch 72) — and this chapter places them side by side and
distinguishes them on a single axis: what question does each one answer, and at what cost.

Cover each signal's actual shape, not just its name. A log is a discrete, timestamped event with
arbitrary structure — expensive per event, but retains full detail. A metric is a numeric,
aggregable time series, cheap to store and query at any timescale precisely because it's
pre-aggregated and low-cardinality — it answers "how much" and "how often," not "why" or "which
specific request." A trace is the causal chain of a single request as it crosses service
boundaries — expensive per request, but it's the only one of the three that can answer "where did
the time go for this one failing request" (Ch 72 owns the mechanics; this chapter only places it in
the taxonomy). Cover cardinality directly as the reason metrics are cheap and logs/traces aren't: a
metric that accidentally includes a high-cardinality label (a user ID, a request ID) stops behaving
like a metric and starts behaving — and costing — like a log, which is the single most common
mistake this chapter should name explicitly.

Cover the actual selection test as the chapter's central deliverable, mirroring Ch 65's approach:
identify the question being asked first — "is the system healthy right now" routes to a metric
dashboard; "did this specific user's request fail, and why" routes to a trace, or a log entry found
via its correlation ID (Ch 21); "what happened around this specific event three hours ago" routes to
a log. None of the three replaces the other two; each is the cheapest tool that can actually answer
its question.

Anchor in real systems: Prometheus's pull-based, low-cardinality time-series model as the dominant
open-source instance of the metrics category, and the cardinality discipline its data model forces
on instrumentation; the "three pillars of observability" framing — logs, metrics, traces — as the
widely used industry vocabulary for this exact taxonomy; Kafka consumer lag as a concrete metric
example, resolving Ch 17's forward reference to Kafka's operational visibility.

Do NOT cover: what specifically belongs in a log or at what level (Ch 69 already covers that in
depth); alerting on any of these three signals (Ch 71); the trace/span data model, context
propagation, or sampling strategy in depth (Ch 72 — this chapter only establishes that traces exist
and what question they answer relative to the other two).
```

---

## Ch 71 — Alerting: Signal vs. Noise

```
This chapter resolves Ch 07 and Ch 68's forward references: what actually triggers a responder to
open the runbook Ch 68 already covers. It is Principle 6 applied to monitoring specifically — an
alert is a process with a real cost (a human's attention, at any hour) and is only justified when
that cost is smaller than the cost of not knowing. This chapter decides which signals from Ch 69 and
Ch 70 clear that bar; it does not cover what the responder does once paged, which is Ch 68's
document, not this chapter's subject.

Cover alert fatigue as the chapter's central failure mode: a high volume of low-value alerts trains
responders to acknowledge and dismiss rather than investigate, so the one alert that actually matters
gets the same reflexive dismissal as the hundred before it that didn't. This is the same
proxy-metric-adjacent failure Ch 49 already named for process generally — the paging system starts
optimizing for "an alert was sent" rather than "a human usefully responded" — applied here to
monitoring rather than issue tracking.

Cover the actionability test as the central selection criterion for whether something should page a
human at all: does this condition have a clear next action and, ideally, a runbook (Ch 68,
referenced, not redefined) — or is it purely informational, in which case it belongs on a dashboard,
not in someone's pocket at 3 a.m. Cover symptom-based versus cause-based alerting as the chapter's
central trade-off and take a position: alert on symptoms a user would actually notice (elevated
error rate, elevated latency) rather than every internal cause that could plausibly produce one (CPU
above a threshold, one replica briefly unhealthy) — a cause-based alert fires on conditions that
often self-correct without user impact, manufacturing exactly the noise the fatigue argument above
describes.

Anchor in real systems: Google's Site Reliability Engineering book, again, as the canonical source
for symptom-based alerting and the practice of separating page-worthy, ticket-worthy, and
log-worthy severity into distinct tiers rather than treating every detected anomaly the same way;
PagerDuty's escalation-policy and on-call-routing conventions as the mainstream tooling
implementation of tiered severity and escalation, referenced here for alert routing specifically,
distinct from the runbook templates Ch 68 already anchored to the same product.

Do NOT cover: the runbook document itself once a responder is paged (Ch 68 already owns that
content); the underlying signal types being alerted on (Ch 69 and Ch 70 already own logs and
metrics; this chapter is only about the decision layered on top of them); the error budget or SLO an
alert is ultimately protecting, or burn-rate alerting mechanics specifically (Ch 73 — referenced as
the target, not derived here).
```

---

## Ch 72 — Distributed Tracing

```
This chapter resolves Ch 07 and Ch 08's forward references by going deep on the one observability
signal that exists specifically because no single service has the whole picture. Ch 08 already
established Little's Law and the risk that local optimization damages global system behavior; a
trace is the direct observational tool for that argument — it's what actually shows a request
queueing at a downstream dependency, not just where a single service spent CPU time.

Cover the trace/span data model precisely: a trace represents one request's full journey; a span
represents one unit of work within it, with parent-child relationships forming a causal tree across
every service the request touched. Cover why this specifically requires a context propagated across
service boundaries — the correlation ID from Ch 21, extended into a full trace context (trace ID,
span ID, parent span ID) that every service in the call chain must read from its inbound request and
forward on every outbound call it makes. A service that doesn't propagate this context doesn't just
lose its own visibility — it breaks the trace for every service downstream of it, which is why this
is a cross-team discipline problem, not a single service's implementation detail.

Cover sampling as the chapter's central cost trade-off, since tracing every request is expensive at
meaningful scale: blanket low-rate sampling (e.g., 1% of all requests) is cheap and simple but can
miss the specific slow or failing request an engineer is actually looking for; tail-based or
error-biased sampling captures requests that were slow or failed with much higher probability, at
the cost of needing to buffer and evaluate a request before deciding whether to keep its trace.

Anchor in real systems: OpenTelemetry as the now-dominant vendor-neutral standard for trace
instrumentation and context propagation across languages and frameworks; Google's Dapper paper as
the foundational, widely cited origin of the trace/span model nearly every modern tracing system
descends from; Jaeger or Zipkin as concrete open-source trace storage and visualization backends
that consume this data in practice.

Do NOT cover: the general logs-versus-metrics-versus-traces taxonomy (Ch 70 already places tracing
within it; this chapter assumes that placement and goes deep on the mechanics); alerting on trace
data or trace-derived latency SLIs (Ch 71); service boundary or coupling design itself (Part II
already owns that — this chapter observes the boundaries Part II designed, it doesn't redesign
them).
```

---

## Ch 73 — Error Budgets and SLOs

```
This chapter closes Part IX by resolving three separate forward references — Ch 07, Ch 62, and
Ch 68 all named this chapter by number — and by naming the target every preceding chapter in this
Part ultimately serves. It connects directly to Ch 01's optimization-target framing and Ch 07's
MTBF/MTTR paradigms: an SLO is the point where "how reliable should this be" stops being a vague
aspiration and becomes a specific number with real consequences attached.

Cover the precise three-term taxonomy as the chapter's foundation, and correct the most common
conflation directly: an SLI (service level indicator) is the thing actually measured — a percentage
of successful requests, a latency percentile; an SLO (service level objective) is the internal
target for that indicator; an SLA (service level agreement) is the external, often contractual
promise, and is typically looser than the internal SLO specifically to leave margin before a missed
internal target becomes a missed external commitment.

Cover the error budget as the direct mechanical consequence of an SLO, and take a position on how it
should be used: a 99.9% availability target isn't just a target, it's an explicit, spendable budget
of allowed failure (roughly 43 minutes a month) — and the budget should be spent deliberately, on
risky deploys, migrations, or experiments, rather than hoarded as if any consumption were a failure.
This reframes reliability work from "always chase more nines" to "spend the budget that's already
been explicitly allowed," which is Principle 6 applied one more time: additional reliability work
past the budget's target is overhead the objective itself says isn't currently justified.

Cover the direct mechanical link to two chapters that already named this one: Ch 62's canary and
progressive-rollout gating decision is driven by burn-rate against this chapter's error budget
(referenced — Ch 62 owns the promotion mechanism itself, not the metric that gates it); Ch 68's
runbook exists specifically to protect this budget by recovering quickly when it's being spent
unexpectedly (referenced, not re-derived).

Anchor in real systems: Google's Site Reliability Engineering book as the canonical origin of the
SLO/error-budget framework and the error-budget-as-a-spendable-resource reframing specifically;
multi-window, multi-burn-rate alerting, documented in Google's SRE workbook, as the concrete
mechanism connecting Ch 71's alerting decision to this chapter's error-budget math — referenced back
to Ch 71 rather than re-explaining alert routing mechanics here; a major cloud provider's published
uptime SLA as a real-world example of the looser, external commitment contrasted directly against a
tighter internal SLO.

Do NOT cover: alert routing and escalation mechanics (Ch 71 already owns how an alert reaches a
human — this chapter owns only the budget math that decides whether one should fire); the
canary/progressive-rollout mechanism itself, including how traffic actually shifts in stages (Ch 62
already owns that; this chapter owns only the signal that gates the decision); the runbook document
a responder follows once paged (Ch 68).
```
