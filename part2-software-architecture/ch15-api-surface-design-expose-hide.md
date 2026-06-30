# Chapter 15 — API Surface Design: What to Expose, What to Hide

**Prerequisites:** [Part I, Ch 03 — Coupling and Cohesion](../part1-systems-thinking/ch03-coupling-and-cohesion.md), [Ch 04 — Abstraction and Information Hiding](../part1-systems-thinking/ch04-abstraction-and-information-hiding.md), [Part II, Ch 13 — Coupling and Cohesion at the Architecture Level](ch13-coupling-cohesion-architecture-level.md), [Ch 14 — Abstraction Layers: When to Add One](ch14-abstraction-layers-when-to-add-one.md). Specifically: information hiding, afferent coupling, and connascence of value.

**New vocabulary introduced:** progressive disclosure

**Key takeaways:**
- Information hiding (Ch 04) applied at the network boundary: every field, parameter, and operation an API exposes is a permanent commitment, cheap to add and exponentially expensive to remove once a consumer depends on it. Design the surface by deciding what you're willing to maintain forever, not by what data happens to be available.
- Minimal surface area means hiding internal state by default and exposing only what a consumer's actual use case requires — never a field because the ORM happened to generate it.
- Progressive disclosure keeps the common path simple while making advanced capability available but not mandatory, so the 90% of consumers who need the basics aren't forced to understand the parameters the 10% need.
- Internal and external APIs carry different stability obligations because their consumers carry different coordination costs: internal consumers can be migrated on a schedule you control; external consumers cannot be coordinated with at all.

---

## The Principle of Minimal Surface Area

**What it is:** Hide all internal state and capability by default; expose only the specific fields a consumer needs to satisfy an actual, named use case.

**Why it exists:** An API surface is afferent coupling (Ch 03) made structural. Expose a field, and some consumer will eventually depend on it — including fields that exist only because an ORM generated them, not because anyone designed them to be public.

**Options:**
1. **Internal state passthrough** — the API serializes and returns the internal domain model or database entity directly
2. **Explicit boundary mapping** — the API returns a dedicated Data Transfer Object (DTO) containing a deliberately reduced subset of the internal data

**Trade-offs:**
- *Passthrough:* fast to write, no mapping boilerplate — but it destroys information hiding outright, coupling every consumer directly to the provider's internal persistence strategy.
- *Boundary mapping:* the database can be refactored freely without touching the API contract, at the cost of a translation layer and duplicated data shapes that have to be kept in sync.

**When to choose each:**
- *Passthrough:* tightly coupled internal scripts or prototypes where the same developer deploys both sides simultaneously.
- *Boundary mapping:* the default for any production API with a consumer outside that developer's direct control.

**Common failure modes:**
- **The leaky database column:** an internal `deleted_at` timestamp, added for auditing, leaks into the public response because the API uses passthrough. A third-party client starts filtering on its presence, and the soft-delete strategy is now permanently load-bearing for an external integration that was never supposed to see it.

**Example:** The POSIX file descriptor API — `open`, `read`, `write`, `close` — is a masterclass in minimal surface area, stable for fifty years precisely because it exposes almost nothing: the consumer deals with plain byte arrays, with the entire complexity of filesystems, block storage, and hardware interrupts hidden underneath. **[Consensus: a field is exposed because a consumer needs it, never because it exists internally]**

---

## Progressive Disclosure

**What it is:** A design strategy where the default request is as simple as possible, and advanced capability is available through optional parameters or expansion mechanisms rather than being present in every response.

**Why it exists:** Most consumers need a small fraction of an API's total capability. Forcing all of them to understand the full surface — including the parameters only power users need — produces cognitive overload and integration errors for everyone.

**Options:**
1. **The god resource** — a single endpoint accepting many parameters, always returning a large, deeply nested payload with every related entity included
2. **Progressive disclosure** — a minimal default response, with related data available via explicit expansion (`?expand=`) or distinct sub-routes

**Trade-offs:**
- *God resource:* minimizes round-trips, but wastes bandwidth on data most callers never use and makes the surface itself hard to document or reason about.
- *Progressive disclosure:* keeps the default fast and the documentation legible, but a client that does need the expanded data either makes multiple calls or has to learn the expansion syntax.

**When to choose each:**
- *God resource:* bulk/asynchronous data-extraction pipelines, where the goal is raw transfer rather than human integration.
- *Progressive disclosure:* the default for any public REST API or SDK.

**Common failure modes:**
- **Configuration bankruptcy:** an endpoint accumulates a dozen-plus optional boolean flags over years (`include_history`, `skip_validation`, `dry_run`), interacting in undocumented and non-deterministic ways, until the endpoint can no longer be safely tested.

**Example:** Stripe's `Charge` object returns a flat, minimal response by default — its `customer` field is just a string ID. A client that needs the full customer record doesn't call a different endpoint; it appends `?expand[]=customer`, and Stripe layers in the nested object only on request. **[Strong Recommendation: make the common case the cheap case, and put everything else behind an explicit ask]**

---

## Internal vs. External API Boundaries

**What it is:** The decision of how aggressively an API is allowed to evolve, determined by whether its consumers are coordinated internal teams or uncoordinated external parties.

**Why it exists:** An API is a contract. A contract between two teams in the same organization can be renegotiated and redeployed together. A contract with an unknown third party cannot be renegotiated at all — it can only be extended or broken.

**Options:**
1. **Aggressive evolution** — fields are deprecated and payloads changed; consumers are expected to update in step
2. **Strict immutability** — nothing is ever removed or behaviorally altered; every change is additive

**Trade-offs:**
- *Aggressive evolution:* the codebase stays clean with no legacy adapter layers — but it requires real-time, synchronous coordination with every downstream caller, which only works when every caller is known.
- *Strict immutability:* integrations written years ago keep working without modification, building durable trust — but the provider absorbs all the resulting accidental complexity, accumulating legacy translation layers indefinitely.

**When to choose each:**
- *Aggressive evolution:* internal service-to-service APIs where every caller is visible and on a coordinated deploy schedule.
- *Strict immutability:* public SaaS APIs, mobile app backends (where you cannot force users to update), and third-party enterprise integrations.

**Common failure modes:**
- **The unannounced deprecation:** a team treats a public API like an internal one and removes a "rarely used" field to simplify a database migration. Every third-party integration depending on that field breaks at once, with no warning and no migration path.

**Example:** GraphQL, designed at Facebook, was built to solve an internal problem — letting a consumer select exactly the fields it needs to avoid over-fetching. Exposed as an *external* API, the same flexibility becomes a liability: the provider now has to support an effectively unbounded set of query shapes, which makes execution-plan optimization and rate limiting against unknown third parties far harder than it is for a fixed internal contract.

---

## Why Smart Engineers Disagree: Consumer-Driven vs. Provider-Driven Surfaces

The most polarizing question in API design is whether the consumer or the provider should dictate the shape of the data returned.

Engineers optimizing for frontend velocity argue for consumer-driven flexibility — typically GraphQL. A rigid REST surface means every new UI field requires a backend ticket and a release cycle; a flexible graph lets the frontend query whatever it needs immediately, without waiting on another team.

Engineers optimizing for backend stability and performance argue for provider-driven control — typically REST or gRPC with a fixed shape. A database cannot efficiently answer arbitrary, deeply nested queries on demand; consumer-driven surfaces tend toward N+1 query disasters and unpredictable load, because the backend has given up the ability to tune execution paths for a known request shape.

This is the Theory of Constraints (Ch 08) applied to API design: if the actual bottleneck is UI iteration speed, consumer-driven flexibility is the right call. If the actual bottleneck is database load or data-access control, provider-driven rigidity is the right call. Exposing flexibility to the consumer always means absorbing the corresponding complexity somewhere in the provider — the only real decision is which side of that boundary is cheaper to pay it on.

*Concepts expanded in later chapters: versioning strategies for evolving an API over time (Part II, Ch 16); the REST vs. RPC vs. event-driven transport choice (Part III, Ch 19); authentication and authorization at the boundary (Part III, Ch 24).*
