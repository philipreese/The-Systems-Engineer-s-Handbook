# Chapter 21 — Error Handling Contracts

**Prerequisites:** [Part I, Ch 07 — Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md), [Part II, Ch 15 — API Surface Design: What to Expose, What to Hide](../part02-software-architecture/ch15-api-surface-design-expose-hide.md), [Ch 16 — Versioning and Backward Compatibility](../part02-software-architecture/ch16-versioning-backward-compatibility.md). Specifically: minimal surface area, backward vs. breaking changes, and partial failure.

**New vocabulary introduced:** correlation ID

**Key takeaways:**
- Error responses are part of the API surface (Ch 15) and carry the exact same backward-compatibility obligations as success responses (Ch 16). An undocumented or inconsistent error shape is a contract violation even when nobody calls it "the contract."
- A status code alone is too coarse to act on. A structured error body — a stable machine-readable code, a human-readable message, and a correlation ID — is what lets a client branch on failure instead of parsing prose.
- The 4xx/5xx boundary isn't a style choice, it's a retry contract: 4xx means the same request will never succeed unchanged, 5xx means it might. Getting this classification wrong doesn't just confuse a human reading logs — it breaks automated retry logic in a specific, predictable direction.
- A single HTTP status code can't represent a batch operation where some items succeeded and others didn't. Partial failure (Ch 07) at the wire level needs a per-item result, not an all-or-nothing verdict pretending the operation was atomic when it wasn't.

---

## Structured Error Responses

**What it is:** Returning a consistent, typed payload for every failure — a machine-readable code, a human-readable message, and a correlation ID — instead of a bare status code or free-form text.

**Why it exists:** A status code alone is too imprecise to act on. A `400 Bad Request` says the client made a structural mistake, but not which field, why, or how to recover programmatically. A structured body closes that gap for any client trying to act automatically rather than just display an error to a human.

**Options:**
1. **Flat string messages** — a human-readable string in the body (`"Invalid email format"`)
2. **Structured error objects** — a stable error code, a descriptive message, and a correlation ID a caller can hand to support or trace internally

**Trade-offs:**
- *Flat strings:* trivial to implement, minimal payload — but completely unactionable by code. A client that wants to handle `insufficient_funds` differently from a generic failure has to regex a human sentence, which breaks the instant the provider rewords it.
- *Structured objects:* decouples the human-readable message from the machine-actionable state — but every specific code or field name emitted becomes a permanent part of the contract (Ch 15) that has to be supported indefinitely.

**When to choose each:**
- *Structured objects:* the default for any machine-to-machine API, public SaaS REST API, or mobile backend.
- *Flat strings:* acceptable only for internal CLI tools or scripts consumed exclusively by a human.

**Common failure modes:**
- **The "200 OK" error:** an exception is caught at the framework level but the response still returns `200 OK` with `{"error": true, "message": "Failed to save"}`. Every piece of standard HTTP-level monitoring, CDN caching, and gateway metrics now reports 100% success while every client is actually broken.
- Returning only a free-form `"error": "something broke"` with no machine-readable classification at all, or an inconsistent error shape from one endpoint to the next.

**Example:** Stripe's error object — a broad `type` (`card_error`), a stable machine `code` (`insufficient_funds`), a human message, and the specific `param` that failed — is the de facto industry reference. RFC 7807 ("Problem Details for HTTP APIs") standardizes the same idea with `type`, `title`, `status`, and `detail` fields, and Google's gRPC error model (`status`, `code`, typed `details`) does the equivalent for strongly typed RPC systems — three different ecosystems converging on the same structural answer. **[Consensus: every production error response needs a stable machine-readable code and a correlation ID, independent of which standard's field names you use]**

---

## The Client vs. Server Fault Domain

**What it is:** The strict segregation of failures into client errors (4xx — the caller's request was wrong) and server errors (5xx — the provider degraded), used as a retry contract, not just a classification label.

**Why it exists:** Whether retrying a failed request can possibly help is a mechanical fact about who's at fault. A 4xx means resending the identical payload will never succeed — the request itself has to change. A 5xx means the provider may have recovered by the time a retry arrives.

**Options:**
1. **Generic error bucketing** — broad `400` for client mistakes, broad `500` for everything else, with the JSON body carrying the real distinction
2. **Semantic fault segregation** — the specific failure maps to a precise status (`409 Conflict`, `429 Too Many Requests`, `502 Bad Gateway`, `503 Service Unavailable`)

**Trade-offs:**
- *Generic bucketing:* less for the backend developer to think about when writing the exception handler — but it pushes the work onto every client, which now has to parse the body just to learn the transport-level nature of the failure, and breaks edge infrastructure that routes on status code alone.
- *Semantic segregation:* aligns directly with standard web infrastructure — a proxy or service mesh can automatically retry a 503 and immediately reject a 400 with zero custom logic — but mapping a rich set of internal domain exceptions onto a rigid, decades-old status code set sometimes feels like forcing a square peg into a round hole.

**When to choose each:**
- *Semantic segregation:* the default for every RESTful boundary and public API.
- *Generic bucketing:* never, for a genuinely resource-oriented API — it isn't a real option, it's the failure mode this section exists to name.

**Common failure modes:**
- **The 500 amplification:** a missing required field isn't caught by validation; the unhandled exception bubbles up and the framework returns a generic `500`. The client's SDK sees a 5xx, assumes transient degradation, and retries the same malformed payload ten times with exponential backoff — turning a simple validation gap into a self-inflicted denial-of-service against the provider.
- Treating every `5xx` as permanent and giving up immediately, or retrying a `429`/validation failure as if it were transient — both invert the contract this section defines.

**Example:** Hitting a rate limit on GitHub's API returns a precise `429 Too Many Requests`, not a generic `400` or `500`. Because the fault domain is explicit, any generic HTTP client or service mesh already knows to back off — no GitHub-specific parsing required.

---

## Partial Failure in Batch Operations

**What it is:** The wire-level representation of a request that mutates many independent items at once, where some succeed and others fail — and a single HTTP status code can't express that split.

**Why it exists:** A batch insert of 100 records that succeeds on 99 and fails on 1 (a duplicate key, say) is exactly the partial failure (Ch 07) distributed systems produce constantly. Forcing one status code onto that result means lying about either the 99 successes or the 1 failure.

**Options:**
1. **All-or-nothing (atomic)** — the whole batch runs in one transaction; any single failure rolls back everything and returns one `4xx`/`5xx`
2. **Per-item result (multi-status)** — the request is accepted, processed as far as possible, and the response lists each item's individual success or failure

**Trade-offs:**
- *All-or-nothing:* absolute consistency and a trivial client mental model — it either fully worked or it didn't — but at scale, the probability that at least one item in a large batch fails approaches 100%, making large atomic batches practically unusable.
- *Per-item result:* maximizes throughput and isolates failures so good data still lands — but it pushes real complexity onto the client, which now has to parse which items failed and assemble a smaller retry batch itself.

**When to choose each:**
- *All-or-nothing:* small, logically inseparable mutations where partial application would be actively wrong (creating an invoice and its line items together).
- *Per-item result:* the default for bulk ingestion, webhooks, and large synchronization jobs where throughput matters more than all-or-nothing simplicity.

**Common failure modes:**
- **The poisoned retry loop:** an all-or-nothing batch of 100 fails on item 45 and returns `400`. A naive client integration just retries the whole batch. Without strict idempotency (Ch 22) on the provider side, items 1–44 get inserted a second time before item 45 fails again — repeating until the database is full of duplicates.
- Returning `200 OK` for a batch but silently dropping the items that failed, so the client has no way to know its data is incomplete.

**Example:** Elasticsearch's `_bulk` API is built entirely around per-item results: because it exists to ingest massive log volumes, failing an entire batch over one malformed document isn't acceptable. It returns `200 OK` (the network request itself succeeded) with an `errors: true` flag and a per-document array of exact status codes and error strings.

---

## Why Smart Engineers Disagree: HTTP Semantics vs. Payload Pragmatism

The most persistent fight over error contracts is between REST purists and RPC pragmatists, and it's really the REST-vs-RPC ontology disagreement from Ch 19 resurfacing at the error layer.

REST advocates treat HTTP status codes as load-bearing infrastructure, not decoration. A `404` is structurally different from a `409`, and proxies, load balancers, and CDNs depend on that distinction to route, cache, and shed load correctly. Ignoring status codes, to them, throws away mechanism the entire internet already provides for free.

RPC and GraphQL advocates see rigid adherence to a fixed, decades-old status code set as accidental complexity (Ch 02) — they'd rather return `200 OK` for every successfully delivered payload and push the real business error entirely into a structured, strongly typed body, arguing this is strictly more expressive than what HTTP's status codes were ever designed to carry.

Both are internally consistent — the failure is mixing them. A system that claims to be a public REST API is structurally bound to honor HTTP semantics; returning `200 OK` with an embedded error body there breaks the contract and the infrastructure built to read it. A system explicitly built as internal RPC over HTTP, by contrast, loses nothing by pushing all error semantics into the payload the way gRPC does — that's a coherent design, not a violation of one. The actual failure mode isn't picking either paradigm; it's refusing to commit to one and ending up with an API that does both unpredictably.

*Concepts expanded in later chapters: retry and idempotency mechanics (Part III, Ch 22); authentication- and authorization-specific error semantics, including the 401-vs-403 distinction (Part III, Ch 24).*
