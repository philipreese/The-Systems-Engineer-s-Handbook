# Ch 67 — API Documentation: What Consumers Need

**Prerequisites:** [Error Handling Contracts](../part03-api-design/ch21-error-handling-contracts.md), [Authentication and Authorization Boundaries](../part03-api-design/ch24-authentication-authorization-boundaries.md), [Internal vs. External API Design](../part03-api-design/ch25-internal-vs-external-api-design.md) (Hyrum's Law), [Keeping Documentation Honest](ch66-keeping-documentation-honest.md)

**New vocabulary introduced:** None — this chapter applies Principle 8 and Hyrum's Law (Ch 25) to API reference content specifically, rather than introducing new concepts.

**Key takeaways:**
- An API consumer is a structurally different reader from every other audience in this Part: often without access to the implementation, without knowledge of internal conventions, and — for an external API (Ch 25) — sometimes outside the organization entirely. The documentation isn't supplemental to the interface; it *is* the interface as far as the consumer is concerned.
- [Strong Recommendation] Generate the reference layer — operations, request and response shapes, types, status codes — mechanically from the interface definition wherever tooling allows. A generated reference cannot drift from what it's generated from the way hand-written prose can. Reserve hand-writing for what generation can't produce: getting-started narrative, integration patterns, and working examples.
- A runnable, copyable example answers "how do I make this request" faster than a paragraph re-describing fields a generated parameter table already shows.
- Hyrum's Law (Ch 25) applies directly to documentation completeness: an incomplete or ambiguous contract doesn't prevent consumers from depending on undocumented behavior — it only hides that dependency from the API owner until a change breaks it.
- An explicit non-guarantee ("iteration order is not defined," "this field may be null") is a documented defense against Hyrum's Law, not just a descriptive courtesy — it's what lets the API owner change the undocumented part later without breaking anyone who was told not to rely on it.

---

Every other chapter in this Part assumes a reader with the source in front of them. This one doesn't. Per Ch 25's internal-vs-external distinction, an API consumer frequently can't inspect the implementation, doesn't know the team's internal conventions, and often can't just ask someone for clarification — which is exactly why API documentation earns separate treatment instead of being folded into the general README case. This chapter covers what that reader needs and how to keep it accurate; it doesn't re-derive how the interface itself should be designed — resource shapes (Ch 20), the error contract (Ch 21), and authentication schemes (Ch 24) are referenced here, not redesigned.

### Decision: Document the Contract, Not the Implementation

**What it is:** API documentation describes the interface's externally observable behavior — what a consumer can rely on — rather than how it's implemented internally.

**Why it exists:** A consumer depends on the contract, not the code behind it. Documenting implementation details invites consumers to depend on things that were never promised and that the API owner is free to change; it also becomes obsolete the moment that internal detail changes, which is more often than the contract itself does.

**Options:** A complete reference covers, for each operation: what it does, its request and response shapes, its authentication requirement (Ch 24, referenced), its error behavior (Ch 21, referenced), and a working example. It does not cover internal algorithms, caching strategy, or other implementation choices that were never part of the promise.

**Trade-offs:** Documenting the contract gives consumers stable expectations and a low support burden, at the cost of needing to be updated alongside interface changes — a cost every option here pays in some form. Documenting implementation detail satisfies curiosity but goes stale fast and actively encourages consumers to depend on things the API owner didn't intend to promise. Minimal documentation is cheap to produce and expensive for every consumer who has to reverse-engineer the gap.

**Common failure modes:** A reference page describes an internal caching layer instead of the response contract a consumer actually integrates against; when the caching strategy changes, the documentation is now wrong about something that was never the consumer's business to know in the first place.

**Example:** GitHub's public API references describe requests, responses, authentication, and observable behavior — never GitHub's internal service implementation. Consumers integrate against the documented contract, which is exactly the boundary Ch 25 draws between an API's public promise and its private implementation.

---

### Decision: Generate the Reference Layer, Write the Narrative Layer by Hand

**What it is:** The chapter's central trade-off — whether the structural parts of an API reference (endpoints, parameters, types, status codes) are produced mechanically from the interface definition or maintained as hand-written prose — and the choice of what to reserve for a human to write.

**Why it exists:** This is Principle 8 again, applied to the most externally visible documentation this handbook covers. Reference content duplicates information that already exists somewhere authoritative — a type definition, an OpenAPI schema, a docstring. Duplicated information drifts (Ch 66); generated documentation removes the duplication by deriving the reference directly from that authoritative source instead of re-describing it by hand.

**Options:**
1. **Generated reference** — derived from OpenAPI/Swagger definitions, source annotations, or docstring tooling (Rustdoc, Javadoc, Sphinx).
2. **Hand-written reference** — endpoint and parameter descriptions maintained independently of the interface definition.
3. **Hybrid** — generate the structural reference; write the narrative layer — getting-started guides, integration patterns, the "why would I reach for this" context — by hand, since generation can't produce it.

**Trade-offs:** Generated reference cannot drift from the signatures it's generated from and costs little to maintain, but it can only describe what exists, not why it's organized that way or how several endpoints combine into a real integration. Hand-written reference allows richer explanation everywhere, at the cost of duplicating information the interface definition already contains and drifting from it the moment either one changes without the other. The hybrid approach pays only the maintenance cost of the narrative layer — which is exactly the kind of external documentation Ch 66 already flagged as prone to drift, because its accuracy depends on a human noticing it's wrong, not a compiler.

[Strong Recommendation] Generate everything the tooling can generate — operations, parameters, request and response schemas, status codes — and write prose only for what generation cannot produce. A working, copyable example usually communicates more than another paragraph of field descriptions the generated table already shows.

**Common failure modes:** A team maintains endpoint descriptions by hand in a separate content system despite an OpenAPI definition already existing. A field is renamed in the schema; the hand-written page is never told. Consumers building against the published documentation get requests rejected against a contract that changed weeks earlier.

**Example:** The OpenAPI specification is the dominant mechanism for generating REST reference documentation directly from a machine-readable contract; GitHub's own REST and GraphQL API references are produced this way, so a schema change propagates to the published docs automatically instead of waiting for someone to notice and update a separate page. Stripe's documentation is the industry's most cited benchmark for the narrative half of this split — precise generated reference paired with realistic, copyable, end-to-end integration examples that answer "how do I actually build this" rather than restating field names a second time.

---

### Decision: Document the Whole Contract, Not Just the Happy Path

**What it is:** Documenting guarantees, limitations, and explicit non-guarantees — not only the success path — as a deliberate defense against Hyrum's Law.

**Why it exists:** Ch 25 established Hyrum's Law: with enough consumers, every observable behavior becomes someone's dependency, whether it was documented or not. An incomplete or ambiguous contract doesn't prevent that — it only hides the dependency from the API owner until a change breaks it. Precise documentation, including explicit statements about what's *not* guaranteed, is what lets an API owner change undocumented behavior later without it counting as a breaking change.

**Options:**
1. **Exhaustive boundary definition** — state ordering guarantees (or their explicit absence), nullability, pagination behavior, rate limits, and the full error contract, not just the 200 response.
2. **Permissive, happy-path-only documentation** — describe the successful case and leave incidental behavior unmentioned.

**Trade-offs:** Exhaustive documentation costs more up front — every boundary has to actually be decided and stated, not left implicit — but it's what protects the API owner's ability to change an internal detail later; a documented non-guarantee makes that change compliant with the contract instead of a break. Permissive documentation is faster to ship initially, but every unstated behavior a consumer happens to observe is a candidate for silent, accidental dependency the API owner won't discover until changing it breaks someone.

**When to choose each:** Exhaustive documentation for any external, multi-tenant, or partner-facing API, where the blast radius of an accidental dependency is largely outside the owner's control. Permissive documentation is defensible only for a genuinely internal, single-team boundary where both sides can coordinate a change directly instead of relying on a documented contract to protect them from each other.

**Common failure modes:** An endpoint happens to return results sorted by insertion order, an artifact of the underlying query rather than a stated guarantee. The documentation says only "returns a list of users." A client builds a UI that assumes that order. Months later an unrelated storage migration changes the underlying query plan, the order shifts, and the client breaks — not because the documented contract changed, but because behavior nobody documented, and therefore nobody promised, was depended on anyway.

**Example:** Java's and Rust's standard library documentation explicitly states that a hash map's iteration order is not guaranteed. That single documented non-guarantee is what lets both languages change their internal hashing implementation across releases without it counting as a breaking change — the explicit absence of a promise is doing real protective work, not just describing an implementation detail.

---

### Why Smart Engineers Disagree

There's little disagreement that the reference layer should be generated — that part of this chapter's position is close to consensus among experienced API designers. The disagreement is about how much hand-written narrative should sit alongside it.

One position keeps hand-written content to a minimum: generated reference plus a handful of working examples is enough, and it's substantially easier to keep accurate than a library of tutorials, since every additional hand-written page is one more thing that can drift (Ch 66) without a compiler to catch it.

The other invests deliberately in getting-started guides, integration walkthroughs, and conceptual narrative, arguing that a successful API is measured by how quickly a new consumer becomes productive, not merely by how complete its field-level reference is — and that gap is exactly what generated reference structurally cannot close.

Both positions accept the same constraint: every hand-written page is a maintenance liability the moment it's published. They differ on how much onboarding value that liability is worth. In practice the answer tracks how cold the audience is — an API consumed entirely by engineers already familiar with the domain gets by on reference and examples; one meant to onboard unfamiliar external developers, the way Stripe's is, earns the narrative investment because the alternative is a support queue answering the same "how do I actually start" question by hand, one ticket at a time.
