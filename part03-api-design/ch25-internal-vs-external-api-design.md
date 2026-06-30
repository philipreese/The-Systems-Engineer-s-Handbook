# Chapter 25 — Internal vs. External API Design

**Prerequisites:** [Part II, Ch 14 — Abstraction Layers: When to Add One](../part02-software-architecture/ch14-abstraction-layers-when-to-add-one.md), [Ch 15 — API Surface Design: What to Expose, What to Hide](../part02-software-architecture/ch15-api-surface-design-expose-hide.md), [Ch 16 — Versioning and Backward Compatibility](../part02-software-architecture/ch16-versioning-backward-compatibility.md), [Part III, Ch 21 — Error Handling Contracts](ch21-error-handling-contracts.md). Specifically: the structural internal/external stability distinction, backward-compatible vs. breaking changes, and the sunset pattern.

**New vocabulary introduced:** Hyrum's Law, consumer-driven contract test

**Key takeaways:**
- Ch 15 drew the structural line between internal and external API stability. This chapter is about what actually changes operationally once that line is crossed: process, tooling, and support obligations, not field exposure.
- What makes an API "external" isn't whether it sits on the public internet — it's whether the provider controls the consumer's deployment schedule. The moment a consumer you can't coordinate with depends on an endpoint, it's functionally external whether or not it was ever designed, documented, or versioned as one.
- Hyrum's Law is the mechanism behind that trap: with enough consumers, every observable behavior — intended or not — eventually becomes something somebody depends on. An undocumented sort order, an incidental field, a timing quirk; none of it is safe to change once enough people are watching.
- SDKs, formal change management, and gateway infrastructure aren't marketing exercises once an API goes external — they're how the cost of that uncoordinated consumer base gets paid for predictably instead of landing, unplanned, on the support queue.

---

## Accidental vs. Intentional Externalization

**What it is:** An API's "external" status is determined by whether the provider controls the consumer's deployment schedule, not by network location. Externalization, once it happens, is a one-way door.

**Why it exists:** The catastrophic case isn't a deliberately published public API — it's an internal endpoint quietly discovered and depended on by a team the provider can't coordinate with. Once that's happened, the endpoint is external in every way that matters, regardless of what anyone intended.

**Options:**
1. **Implicit internal APIs** — built fast, without formal versioning or documentation, on the assumption every caller lives in the same repository and can be updated in lockstep
2. **Formally externalized APIs** — hardened with explicit documentation, SLAs, and real backward-compatibility guarantees, because the provider has no way to force a consumer to migrate

**Trade-offs:**
- *Implicit internal:* maximizes velocity — fields get renamed, endpoints deleted, with nothing to stop it — but guarantees a severe outage the moment an uncoordinated team turns out to be depending on exactly the thing that just changed.
- *Formally externalized:* protects every consumer and builds real organizational trust — but imposes a real procedural tax, permanently slowing how fast the underlying code can move.

**When to choose each:**
- *Implicit internal:* only when provider and consumer ship through the exact same pipeline, owned by the exact same team.
- *Formally externalized:* the moment an API crosses an organizational boundary where the provider can't simply refactor the caller's code itself.

**Common failure modes:**
- **The Hyrum's Law trap:** an internal API happens to return users sorted alphabetically — purely an accident of the underlying database index, never documented or intended. A partner team discovers the endpoint and builds a UI that assumes that ordering. Two years later the index changes, the sort order randomizes, and the partner's UI breaks — the provider never agreed to a contract about sort order, but Hyrum's Law made one anyway: with enough consumers, every observable behavior eventually becomes something somebody depends on, whether it was ever documented or not.

**Example:** Google engineers fight this at scale by design: an internal service can accumulate thousands of uncoordinated consumers simply because the company is large enough. Google's infrastructure actively randomizes hash-map iteration order and protobuf field serialization at runtime — deliberately making the *unspecified* parts of the output chaotic, so consumers are structurally forced to depend only on the explicit contract, because nothing implicit is stable enough to depend on by accident. **[Strong Recommendation: treat "discovered and depended on by someone you can't coordinate with" as the actual definition of external, not "published on purpose"]**

---

## Formal Change Management

**What it is:** The procedural infrastructure — changelogs, deprecation notices, consumer-driven contract tests — that replaces direct coordination once a provider can no longer just walk over and ask a consumer to update their code.

**Why it exists:** An internal team can run one atomic commit that updates both the API and every caller of it. An external consumer has to be notified, persuaded, and given real time to migrate — there's no commit that touches both sides at once.

**Options:**
1. **Synchronous code updates** — the provider changes the API and the consumer's code in the same commit
2. **Formal change management** — consumer-driven contract tests verify changes don't break known clients, changelogs are published, and deprecation notices go out well before any structural change ships

**Trade-offs:**
- *Synchronous updates:* instantaneous and provably safe, with no legacy version to maintain — but structurally impossible the moment provider and consumer cross an organizational boundary.
- *Formal change management:* allows genuinely decoupled evolution on both sides — but it's slow. It requires real documentation discipline and running multiple API versions simultaneously through a real sunset window (Ch 16).

**When to choose each:**
- *Synchronous updates:* tight microservice clusters living in a single monorepo.
- *Formal change management:* the default for any API with third-party or autonomous-partner consumers.

**Common failure modes:**
- **The ghost deprecation:** a field gets removed because internal dashboards show nobody using it — but those dashboards never tracked field-level usage in the first place. With no consumer-driven contract test and no published changelog, thousands of integrations break silently and the support queue is overwhelmed before anyone notices the pattern.

**Example:** GitHub's API treats this as a real operational cost, not paperwork: exhaustive changelogs for even minor behavioral tweaks, explicit deprecation headers, and scheduled brownouts (deliberate, brief failures) that force lagging consumers to notice and migrate before the underlying endpoint is actually deleted — the same sunset discipline from Ch 16, applied at the process level instead of just the wire level.

---

## The Stability Buffer: SDKs vs. Raw Wire

**What it is:** Whether the provider distributes and maintains native client libraries that wrap the API, or leaves every consumer to integrate against the raw HTTP/JSON or gRPC contract directly.

**Why it exists:** An SDK is an anti-corruption layer (Ch 14) deployed straight into the consumer's own codebase. If the provider changes pagination from an integer offset to an opaque cursor (Ch 23), the SDK absorbs that translation internally — the consumer's `list_users()` call never has to change.

**Options:**
1. **Raw wire contracts** — an OpenAPI/gRPC spec is published; consumers write their own clients, retry logic, and pagination loops
2. **Published SDKs** — the provider builds and maintains native libraries across languages that wrap the raw calls

**Trade-offs:**
- *Raw wire contracts:* costs the provider almost nothing to maintain — but pushes every bit of distributed-systems complexity (retries, backoff, token refresh) onto each consumer individually, multiplying the same integration bugs across every client that writes its own version.
- *Published SDKs:* an excellent developer experience, and lets the provider ship performance or security fixes transparently — but means maintaining parallel codebases across however many languages, multiplying the cost of every new feature by however many SDKs exist.

**When to choose each:**
- *Raw wire contracts:* internal RPC where a platform team already code-generates stubs centrally, or low-stakes single-purpose integrations.
- *Published SDKs:* public SaaS APIs where developer experience is a genuine competitive differentiator.

**Common failure modes:**
- **The out-of-sync SDK:** the backend ships a new endpoint, but the team maintaining the Ruby SDK is backlogged. Ruby consumers can't reach the new feature for months — fragmenting the ecosystem and pushing frustrated developers to bypass the SDK and write raw HTTP calls anyway, defeating the entire reason it existed.

**Example:** An internal gRPC service needs zero public documentation — a compiled `.proto` file is the entire contract for callers who are all internal anyway. Stripe, by contrast, maintains official SDKs across seven languages and treats the SDK, not the underlying REST endpoints, as the actual product — using it to hide the genuinely terrifying complexity of payment retries and idempotency (Ch 22) behind a clean local interface.

---

## External Edge Infrastructure

**What it is:** The gateways, rate limiters, and developer portals that exist specifically to manage untrusted, uncoordinated third-party traffic — infrastructure internal services rarely need at all.

**Why it exists:** Internal services can lean on a lightweight service mesh and tribal knowledge about who's calling whom. External traffic needs strict policing, automated credential issuance, and self-serve documentation, because there's no Zoom call to schedule with every new third-party developer.

**Options:**
1. **Internal service mesh** — lightweight sidecar proxies (Envoy) handling routing and mTLS between trusted internal nodes
2. **Public API gateway + developer portal** — a dedicated edge proxy (Kong, Apigee) terminating external traffic, enforcing rate limits, validating third-party credentials, and feeding a self-serve portal

**Trade-offs:**
- *Internal mesh:* low latency, low operational overhead — but has no mechanism for throttling an abusive external tenant or onboarding a stranger without human intervention.
- *Gateway + portal:* fully automates external onboarding and protects the backend from traffic spikes it never agreed to — but introduces a real new single point of failure and a dedicated platform team to run it.

**When to choose each:**
- *Internal mesh:* all inter-service traffic within the trusted boundary.
- *Gateway + portal:* the moment any third party can interact with the system without a human approving it first.

**Common failure modes:**
- **The unthrottled onboarding:** a public API launch points external traffic directly at the internal load balancer with no gateway in front of it. One external developer's buggy `while (true)` loop generates 50,000 requests per second, with no tenant-level rate limit anywhere to stop it — and the resulting database overload takes down the core product for every customer, not just the one with the bad script.

**Example:** Twilio and SendGrid run entirely on self-serve gateway infrastructure: a new developer creates an account, the portal issues scoped API keys automatically, and the gateway starts metering traffic against a billing tier immediately — no engineer on the provider's side ever has to be involved in onboarding a single customer.

---

## Why Smart Engineers Disagree: The True Cost of Developer Experience

The real fight over external APIs isn't about the JSON shape — it's about how much to invest in everything around it.

Engineers optimizing for backend efficiency argue a published OpenAPI spec is enough: "we provide the accurate contract; how you consume it is your problem." SDKs, polished developer portals, and detailed changelogs look to them like marketing spend competing with actual engineering for the same budget.

Engineers optimizing for adoption argue an API with no native SDKs and no real documentation is effectively unusable, and that the engineering cost of maintaining an SDK is far cheaper than the human cost of fielding thousands of support tickets from developers who got HMAC verification or cursor pagination subtly wrong on their own.

Going external is a product decision wearing a technical one's clothes, and that's what resolves the disagreement: the developer experience *is* the product once an API is public. Skipping SDKs, changelogs, and a portal doesn't make that complexity disappear — it just moves the cost off the engineering budget and onto the support queue, where it's harder to plan for and more expensive per incident. If the organization isn't prepared to fund that infrastructure, it isn't actually prepared to go external — internal APIs are optimized for change; external ones are optimized for continuity, and continuity has a real, ongoing price.
