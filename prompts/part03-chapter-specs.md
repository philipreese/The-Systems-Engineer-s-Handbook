# Part III — Chapter Specifications

Pre-filled special instructions for each Part III chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part III: Part I and Part II. Readers are expected to know the full Part I
vocabulary plus Part II's: information hiding, minimal surface area, progressive disclosure,
internal vs. external API stability, backward compatibility and versioning, temporal
coupling, the synchronous vs. asynchronous coupling decision, and the saga pattern. Part III
does not re-derive any of these — it applies them to the specifics of API wire design.

Three forward references already exist from Part II and must be honored here:
- Ch 15 promised that the REST vs. RPC vs. event-driven transport choice is covered in Ch 19,
  and that authentication/authorization at the boundary is covered in Ch 24.
- Ch 16 promised that REST vs. RPC transport mechanics are covered in Ch 19.
- Ch 17 promised that idempotency key mechanics are covered in depth in Ch 22, and
  specifically name-checked "a unique idempotency key enforced with a database UNIQUE
  constraint" as the worked example — Ch 22 should deliver on that exact mechanism.

---

## Ch 19 — REST vs. RPC vs. Event-Driven

```
Part II Ch 17 already decided synchronous vs. asynchronous as a coupling question. This
chapter is a different decision: once you know whether a call is sync or async, what shape
does the call itself take on the wire? REST, RPC, and event-driven are not interchangeable
defaults — each encodes a different mental model of what an API is for.

Cover REST's resource model: nouns with identity and state, manipulated through a small,
uniform verb set (the HTTP methods). Cover why HATEOAS is the textbook ideal almost no real
API implements, and why that's a reasonable trade rather than a failure.

Cover RPC (gRPC, Thrift, Twirp): action-oriented, strongly typed via a schema (protobuf),
optimized for performance and codegen rather than human URL legibility. Cover why RPC tends
to win for internal service-to-service calls and REST tends to win for public APIs consumed
by many unrelated clients.

Cover event-driven as a third style with its own shape (a fact published, not a request
made) — but do not re-derive the sync/async coupling argument from Ch 17. Here the focus is
purely on message/event shape and contract, assuming the decision to go async is already made.

Anchor in real systems: gRPC at Google's internal scale (protobuf contracts, HTTP/2
multiplexing), Stripe and GitHub's REST APIs as resource-oriented public APIs, Twirp as a
pragmatic RPC-over-HTTP bridge, Kafka event schemas as the event-driven case.

Do NOT cover: the sync vs. async coupling decision itself (Ch 17 — assume it's already made),
minimal API surface area (Ch 15), resource modeling in depth (Ch 20), idempotency (Ch 22),
pagination (Ch 23), authentication and authorization (Ch 24). This chapter is about which
wire-level style to choose, not what to put in it or how to secure it.
```

---

## Ch 20 — Resource Modeling

```
Once REST (or a resource-shaped RPC API) is chosen, this chapter covers how to actually model
the resources themselves. A resource is a noun with identity and state — not a remote
procedure call wearing a URL.

Cover URI design principles: model around domain nouns the consumer cares about, not
internal database tables or implementation structures. Cover hierarchical nesting and where
it should stop — deep nesting (more than ~2 levels) is a common failure mode that couples the
URL structure to a specific traversal path instead of letting resources stand on their own.

Cover "the action problem" directly: many real operations don't map cleanly to CRUD
("publish a post," "cancel an order," "refund a charge"). Cover the patterns for handling
this honestly — a sub-resource action endpoint (`POST /orders/{id}/cancel`) versus a
shapeless generic command endpoint — and why the former preserves the resource model while
the latter quietly turns REST into RPC with extra steps.

Cover modeling relationships between resources: embedding related data directly versus
returning a link/reference and requiring a second request, and the trade-off each implies
for client round-trips versus payload bloat.

Anchor in real systems: Stripe's `PaymentIntent` as a resource with an explicit lifecycle
state machine rather than a pile of action verbs, GitHub's nested resource hierarchy
(`/repos/{owner}/{repo}/issues`), "POST as the verb of last resort" as the working convention
for actions that don't fit CRUD.

Do NOT cover: the REST vs. RPC vs. event-driven choice itself (Ch 19), minimal surface area
and information hiding (Ch 15), pagination of resource collections (Ch 23), error response
shape (Ch 21). This chapter is about resource shape, not transport choice, field exposure,
or failure reporting.
```

---

## Ch 21 — Error Handling Contracts

```
Error responses are part of the API surface (Ch 15) and carry the same backward-compatibility
obligations as success responses (Ch 16) — an undocumented or inconsistent error shape is a
contract violation even when nobody calls it "the contract."

Cover structured error response design: a machine-readable error code, a human-readable
message, and a correlation/trace ID a caller can hand back to support. Cover HTTP status
code semantics and the common failure mode of overloading status codes incorrectly (returning
200 with an error body, or using a single generic 400/500 for everything).

Cover the client-error vs. server-error distinction precisely: 4xx means the caller's request
was wrong and retrying unchanged won't help; 5xx means the provider failed and retrying may
be safe. Getting this classification wrong breaks every consumer's retry logic.

Cover partial failure in batch and bulk endpoints: when some items in a request succeed and
others fail, a single all-or-nothing status code cannot represent reality — cover the
per-item result pattern as the honest alternative.

Anchor in real systems: Stripe's error object (`type`, `code`, `message`, `param`), RFC 7807
"Problem Details for HTTP APIs" as a real industry convergence point, Google's API error
model (`status`, `code`, `details`) as the gRPC-side equivalent.

Do NOT cover: idempotency and retry mechanics (Ch 22 — this chapter defines what an error
looks like, not what a client should safely do in response), the 401-vs-403 distinction in
depth (Ch 24 covers that specifically), versioning of the error shape itself beyond noting
it follows the same rules as Ch 16, general fail-fast/reliability principles at the system
level (Part I, Ch 07, already covered — this chapter is about the wire-level contract only).
```

---

## Ch 22 — Idempotency

```
Part II Ch 17 established that at-least-once delivery is the realistic default for
asynchronous systems, and explicitly deferred the mechanics of surviving it to this chapter.
This chapter delivers on that: idempotency is what makes at-least-once delivery survivable.

Cover natural idempotency first: some operations are idempotent by construction (PUT,
DELETE — repeating them produces the same end state) while others are not (POST, by
default, creates a new thing every time it's called). Cover why this distinction is not
optional trivia — it determines whether a network retry is safe by default.

Cover designed idempotency in depth: the idempotency key pattern. A client generates a
unique key per logical operation and sends it with the request; the server uses a
uniqueness constraint to detect a duplicate and returns the original cached response instead
of re-executing the operation. Cover the key-expiry/retention window problem: how long the
server must remember a key, and what happens to a retry that arrives after the window closes.

Cover idempotency as the practical, achievable substitute for exactly-once delivery, which
Ch 17 established is not realistically achievable across a real network boundary. Idempotency
doesn't prevent duplicate delivery — it makes duplicate delivery harmless.

Anchor in real systems: Stripe's `Idempotency-Key` header as the canonical real-world
implementation, a PostgreSQL `UNIQUE` constraint on the idempotency key column as the
deduplication mechanism, payment processing as the textbook case where duplicate execution
(a double charge) is catastrophic rather than merely annoying.

Do NOT cover: at-least-once vs. exactly-once delivery semantics in general (Ch 17 already
covers this — assume the reader knows it and go straight to the mechanism), the saga pattern
and compensating actions (Ch 17), general error response shape (Ch 21). This chapter is
specifically about making retried requests safe, not about distributed transactions or error
formatting.
```

---

## Ch 23 — Pagination and Streaming

```
This chapter covers how an API returns a collection too large for one response. Pagination
is a consistency and performance contract, not a UI convenience — the strategy chosen
determines whether results stay correct under concurrent writes and how expensive a query is
at depth.

Cover offset/limit pagination: simple to implement and reason about, but breaks under
concurrent inserts or deletes (page drift — an item shifts pages between requests) and gets
measurably slower at depth, because the database still has to scan and discard every row
before the offset.

Cover cursor-based (keyset) pagination: stable under concurrent writes, with consistent
performance regardless of depth, at the cost of requiring a sortable, unique cursor field
and giving up arbitrary "jump to page N" access.

Cover streaming as the alternative for very large or genuinely open-ended result sets —
newline-delimited JSON, Server-Sent Events, or gRPC server streaming — where pagination
itself stops being the right mental model.

Anchor in real systems: Stripe's cursor-based pagination (`starting_after`/`ending_before`),
GitHub's `Link` header pagination (RFC 5988), the offset-pagination performance cliff on a
large PostgreSQL table (`OFFSET` requires scanning and discarding every preceding row), a
firehose-style streaming API as a case where pagination doesn't apply at all.

Do NOT cover: resource modeling itself (Ch 20), the sync vs. async or REST vs. RPC transport
decisions (Ch 17, Ch 19), database query optimization in general (Part XII). This chapter is
about how a collection is returned, not how it's modeled or how the underlying query is
tuned.
```

---

## Ch 24 — Authentication and Authorization Boundaries

```
Ch 15 promised this chapter would cover authentication and authorization at the API
boundary. This chapter's job is architectural placement — where these decisions get
enforced — not protocol mechanics or cryptography, which belong to Part XI (Security).

Cover the authn-vs-authz distinction briefly at the boundary level (who are you, vs. what
are you allowed to do) — a full treatment is Part XI Ch 82's job; here it's enough to
establish that they are different concerns enforced at potentially different layers.

Cover where enforcement happens: centralized at an API gateway versus independently in each
service, and the trust-boundary question this creates — does an internal service re-validate
an identity on every call, or trust that an upstream gateway already did? Cover the
token-passing pattern: a signed token (commonly a JWT) issued once and propagated downstream,
verified at each hop versus trusted blindly within a network perimeter.

Cover the architectural failure mode directly: the confused-deputy problem, where a service
trusts the network perimeter alone and is exploited through a compromised or careless
neighbor on the same internal network. Frame zero-trust (verify at every hop, not just the
edge) as the structural response to this failure mode.

Anchor in real systems: OAuth 2.0 / OIDC as the dominant delegated-auth token flow, API
gateways (Kong, Envoy) performing centralized authentication, a confused-deputy incident
pattern as the concrete failure case.

Do NOT cover: cryptographic mechanics of specific auth protocols, threat modeling, secrets
management (Part XI, Ch 79–84 — those go deep on the security side), specific authorization
models like RBAC/ABAC beyond naming them, general API surface design (Ch 15). This chapter
answers where authn/authz boundaries are drawn architecturally, not how the protocols
themselves work.
```

---

## Ch 25 — Internal vs. External API Design

```
Ch 15 already established the structural distinction between internal and external API
stability obligations. This chapter goes deeper into what changes operationally once an
API actually has external consumers — process and tooling, not just the field-exposure
decision Ch 15 covered.

Cover what governance changes once consumers can no longer be coordinated directly: formal
change-management process, consumer-driven contract testing, SDKs as a stability buffer that
absorbs wire-format changes so the consumer's code doesn't have to, and the documentation and
support obligations that only start to matter at the point of external exposure.

Cover the trap of accidentally "going external": an API built and intended as internal-only
gets discovered and depended on by a party outside the team's control, and is now functionally
external whether or not it was ever versioned, documented, or hardened as one. Name Hyrum's
Law directly: with enough consumers, every observable behavior of a system — intended contract
or not — will eventually be depended on.

Cover the operational infrastructure that tends to only show up once an API goes external:
API gateways, developer portals, published changelogs and deprecation notices.

Anchor in real systems: Hyrum's Law (Google) as the governing principle, GitHub's investment
in API documentation, SDKs, and deprecation notices as the visible cost of going external, a
direct contrast between an internal RPC service with no public docs and a published SDK-backed
public API for the same underlying capability.

Do NOT cover: re-deriving the structural internal/external distinction itself (Ch 15 already
defined it — reference it, don't repeat it), versioning mechanics (Ch 16), authentication and
authorization (Ch 24). This chapter is about what changes in process and tooling, not about
what changes in the contract itself.
```

---

## Ch 26 — FFI and Native Binding Design

```
This chapter is a deliberate change of register from the rest of Part III: not a network API,
but a language-boundary API — the interface used when code in one language calls into code
written in another (Python calling C, Rust exposing a C ABI, JNI for Java/native interop).

Make the central argument explicit: an FFI boundary is information hiding (Part I, Ch 04)
under far harsher constraints than a network API. There's no garbage collector watching the
other side, no shared type system across the boundary, and a crash on one side takes down the
entire process — unlike a network call, which fails gracefully and can be retried (Ch 22).
Treat it as the least forgiving API surface covered in this book.

Cover the ABI-vs-API distinction: an FFI boundary commits to a binary calling convention,
not just a logical contract — changing a struct's field order is a breaking change even if
no "interface" technically changed.

Cover ownership and memory lifetime across the boundary as the central design problem: who
allocates, who frees, and what happens if either side gets it wrong. This is the FFI-specific
version of the resource-ownership question, and it has no automatic answer the way it does
within a single managed-memory language.

Cover the C ABI as the de facto universal lowest common denominator nearly all cross-language
interop is built on, even between two languages that are neither one C. Cover error handling
across the boundary: exceptions generally cannot cross an FFI call, so the common patterns are
error codes or out-parameters instead.

Anchor in real systems: CPython's C API and the GIL as a concrete, well-documented real
example, Rust's `unsafe extern "C"` blocks and the fact that the Rust standard library
explicitly does not guarantee a stable ABI of its own — everything that needs to be stable
goes through the C ABI instead — SQLite's C API as a deliberately minimal, decades-stable FFI
surface that nearly every language binds to.

Do NOT cover: general API surface design principles already established in Ch 15 (reference
them as the network-API analogue rather than re-deriving them), the broader practice of
writing unsafe or low-level code (Part IV, Ch 33 covers that), concurrency and threading
concerns across a language boundary (Part X). This chapter is about the shape of the
boundary itself, not the implementation technique on either side of it.
```
