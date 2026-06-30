# Chapter 19 — REST vs. RPC vs. Event-Driven

**Prerequisites:** [Part II, Ch 15 — API Surface Design: What to Expose, What to Hide](../part02-software-architecture/ch15-api-surface-design-expose-hide.md), [Ch 17 — Synchronous vs. Asynchronous Communication](../part02-software-architecture/ch17-sync-vs-async-communication.md). Specifically: minimal surface area, temporal coupling, the synchronous vs. asynchronous coupling decision, and the saga pattern.

**New vocabulary introduced:** HATEOAS, event-carried state transfer

**Key takeaways:**
- Ch 17 decided whether an interaction is synchronous or asynchronous. This is the next layer down: once that's settled, what shape does the call take on the wire? REST, RPC, and event-driven are not stylistic variants of the same idea — they're different ontologies for what an API *is*.
- REST models a system as nouns with state, manipulated through a small uniform verb set. RPC models it as functions to invoke. Event-driven models it as a stream of facts that already happened. Each assumption shapes coupling, evolvability, and debuggability differently.
- The dominant real-world failure isn't picking the wrong ontology — it's mixing them unintentionally: verb-shaped endpoints wearing REST's clothes, resource-shaped services wearing RPC's clothes, or events that are secretly commands in disguise.
- HATEOAS is REST's textbook ideal and almost nobody implements it in production, for good reason: the coordination tax it imposes rarely buys back more than disciplined, documented, pragmatic REST already provides.

---

## REST: Resource-Oriented Architecture

**What it is:** A system modeled as nouns (resources) identified by stable URIs, with state mutated through a small, uniform set of verbs — the HTTP methods (GET, POST, PUT, PATCH, DELETE).

**Why it exists:** REST leverages the mechanics the web already provides for free. Mapping domain operations onto standard HTTP methods lets CDNs, gateways, and reverse proxies cache reads, rate-limit writes, and route traffic without understanding anything about the application underneath them.

**Options:**
1. **Pragmatic REST** — resources identified by URLs, manipulated via HTTP verbs, with clients relying on out-of-band documentation (OpenAPI/Swagger) rather than in-band discovery
2. **Strict HATEOAS** (Hypermedia As The Engine Of Application State) — the server embeds the actions currently available directly in each response, so the client never needs prior knowledge of the URL structure

**Trade-offs:**
- *Pragmatic REST:* legible to humans, universally supported, highly cacheable — but rigidly mapping everything to a noun forces complex business transactions ("approve a loan and transfer the funds") into manufactured resources like `POST /loan-approvals` that don't map cleanly to any real domain noun.
- *Strict HATEOAS:* theoretically decouples the client from the server's URL structure entirely, letting routes change without breaking clients — but in practice it bloats every payload with link metadata, breaks static client code generation, and imposes a coordination cost almost no team recovers in exchange.

**When to choose each:**
- *Pragmatic REST:* the default for public-facing APIs, mobile backends, and third-party integrations, where human discoverability and ecosystem trust matter more than theoretical client/server decoupling.
- *Strict HATEOAS:* almost never. The computational and cognitive tax outweighs the decoupling benefit in nearly every real system.

**Common failure modes:**
- **HTTP verb abuse:** mapping a destructive, non-idempotent action onto GET — `GET /users/123/delete` — so a browser or edge cache that pre-fetches the URL for performance accidentally executes a deletion.
- **Verb endpoints disguised as resources:** `/createUser`, `/resetPassword` — URLs that are really RPC calls wearing REST's clothing, losing REST's uniform-interface benefit without gaining RPC's typing benefit.
- **The god resource:** a single endpoint that returns an entirely different shape depending on query parameters, to avoid a second round trip — destroying the predictable, cacheable resource model REST exists to provide.

**Example:** Stripe and GitHub are the canonical cases of pragmatic REST done well: vast, complex domains exposed through predictable, noun-based URIs, explicitly versioned and heavily documented, with no real HATEOAS in sight. GitHub's API includes a thin layer of hypermedia links for pagination and navigation — partial adoption, not the strict ideal — which is close to the practical ceiling most real APIs ever reach. **[Strong Recommendation: pragmatic REST over strict HATEOAS for any API with consumers outside your direct coordination]**

---

## RPC: Procedure-Oriented Architecture

**What it is:** A system modeled as functions a client invokes on a remote server as if it were local — explicit, specific verbs (`CalculateCompoundInterest()`) rather than REST's noun-based limitations.

**Why it exists:** To optimize for computational performance, strict typed contracts, and code generation, at the expense of human legibility — the opposite trade REST makes.

**Options:**
1. **gRPC** — protobuf binary serialization over multiplexed HTTP/2, Google's high-performance RPC framework
2. **Twirp** — protobuf's strict schema definitions transmitted over plain HTTP/1.1, trading gRPC's streaming for ordinary load-balancer compatibility

**Trade-offs:**
- *gRPC:* fast, heavily compressed, and enforces backward/forward compatibility at compile time — but the binary format kills naive `curl` debugging and requires layer-7 load balancers that actually understand multiplexed HTTP/2 streams.
- *Twirp:* keeps protobuf's compile-time safety and codegen while routing through completely ordinary infrastructure (standard ALBs) — but gives up gRPC's bidirectional streaming.

**When to choose each:**
- *gRPC:* the default for high-throughput, internal service-to-service communication where serialization cost is actually on the critical path.
- *Twirp:* strict schema enforcement is wanted, but the infrastructure team isn't ready to operate a full HTTP/2 service mesh.

**Common failure modes:**
- **The local-method illusion:** generated client stubs (`client.FetchUser(id)`) look exactly like a local function call, so a developer puts one inside a tight loop, forgets it crosses a real network boundary, and collapses system throughput under an N+1 latency tax they never noticed introducing.
- **REST-shaped RPC:** a gRPC service designed as `UserService.GetUser(UserIdRequest)` — overloaded into acting like a resource container instead of a procedure boundary, losing RPC's clarity without gaining any of REST's uniformity.
- Overusing fine-grained RPC calls until the system becomes chattier across the network than it ever was as a monolith.

**Example:** Google's internal Stubby — gRPC's predecessor — proved at hyperscale that parsing JSON strings on every call is an unacceptable tax on CPU. Enforcing binary protobuf RPCs internally meant every service talks through a strictly typed, compiled contract instead of a loosely typed string format. **[Consensus: gRPC for internal service-to-service traffic where both ends are under your control; REST where they aren't]**

---

## Event-Driven: Fact-Oriented Payloads

**What it is:** Messages placed on an asynchronous broker that declare an immutable fact — "this already happened" — rather than a command. Where REST and RPC are inherently request-shaped, events are fact-shaped.

**Why it exists:** To achieve real temporal decoupling (Ch 17): the publisher doesn't know and doesn't need to know who consumes a fact or what they do with it. This is event-driven decoupling (Ch 13) expressed as a payload design, not just a transport choice.

**Options:**
1. **Thin events** — the payload carries only an identifier and an event type (`{"event": "OrderPlaced", "order_id": "123"}`); the consumer fetches the full state itself
2. **Fat events** (event-carried state transfer) — the payload carries the full state of the resource at the moment of the event

**Trade-offs:**
- *Thin events:* lightweight, and the consumer never processes stale data because it always fetches current state — but that fetch reintroduces exactly the temporal coupling events were meant to eliminate; if the publisher's API is down, the consumer can't process the event at all.
- *Fat events:* eliminates temporal coupling entirely — the consumer has everything it needs without ever calling back to the publisher — but risks eventual-consistency collapse: if delivery lags, a consumer can act on a snapshot that's already been superseded by a newer mutation it hasn't seen yet.

**When to choose each:**
- *Thin events:* the payload would be very large (video processing pipelines), or privacy/regulatory constraints prohibit broadcasting full record contents onto a shared bus.
- *Fat events:* the default for CQRS read projections (Ch 18) and saga orchestration (Ch 17), where cross-domain data availability matters more than payload size.

**Common failure modes:**
- **Commands disguised as events:** publishing `SendWelcomeEmail` to a topic looks like an event but is really an RPC command wearing an event's name — the publisher is now tightly coupled to the email service's existence, with none of the decoupling events were supposed to buy. The fact-shaped version is `UserRegistered`; whether to send a welcome email is the email service's own decision to make.
- **Event explosion:** every internal state change becomes its own published event with no curation, until ambiguous names (`UserUpdated` meaning a dozen different things to a dozen different consumers) make the event stream harder to reason about than the calls it replaced.

**Example:** Kafka event schemas in heavily decoupled architectures lean toward fat events modeled in Avro or protobuf — a `PaymentSecured` event carries full transactional metadata, letting fulfillment, analytics, and ledger services each operate entirely independently of the billing API's uptime.

---

## Why Smart Engineers Disagree: Nouns vs. Verbs in Complex Domains

The sharpest friction in wire-level API design shows up modeling a complex, multi-step business transaction — and it's really a disagreement about which side of the network boundary you're standing on.

REST advocates insist everything be modeled as a noun, even a multi-step process. A checkout that needs fraud checks, inventory verification, and payment goes through an abstract `CheckoutSession` resource, mutated via `PATCH` requests as it progresses. They argue this preserves the predictability of web infrastructure and gets standard HTTP caching and conditional-request semantics for free.

RPC advocates see this as architectural gymnastics. A checkout is fundamentally an action, not an entity — forcing a verb to masquerade as a noun produces an opaque state machine instead of an obvious one. They'd rather expose `ExecuteCheckout()` directly and make the execution path explicit to whoever's calling it.

Both are right, just on different sides of the perimeter. Public interfaces need REST's stability and the web infrastructure that comes for free with it — that's the audience that can't be coordinated and needs a contract that ages well. Internal services, coordinating procedural sagas across boundaries you control, lose nothing by using RPC verbs and gain real clarity by not mapping every process onto an invented noun. The mistake isn't choosing REST or RPC — it's applying the public-interface argument to an internal boundary, or the internal-clarity argument to a public one.

REST is a state interface. RPC is a function boundary. Events are a recorded history multiple systems interpret independently. None of the three are competing implementations of the same abstraction — and the actual failure mode in this chapter is treating them as if they were.

*Concepts expanded in later chapters: resource modeling in depth (Part III, Ch 20), idempotency mechanics (Part III, Ch 22), pagination strategies (Part III, Ch 23), authentication and authorization at the boundary (Part III, Ch 24).*
