# Chapter 16 — Versioning and Backward Compatibility

**Prerequisites:** [Part I, Ch 03 — Coupling and Cohesion](../part1-systems-thinking/ch03-coupling-and-cohesion.md), [Part II, Ch 15 — API Surface Design: What to Expose, What to Hide](ch15-api-surface-design-expose-hide.md). Specifically: afferent coupling and the minimal-surface-area principle.

**New vocabulary introduced:** sunset pattern

**Key takeaways:**
- Once an API has a consumer, every change is either backward compatible or breaking — there is no neutral middle category, only different amounts of coordination cost. Most versioning incidents are classification failures: an engineer believed a change was safe when it wasn't.
- A version number in a URL does not make an API evolvable. A disciplined process for classifying changes and migrating consumers off old contracts does. Versioning is an operational discipline, not a structural feature.
- URI versioning, header-based versioning, and schema-level versioning (protobuf, Avro) all solve the same problem — running multiple contracts at once — with different trade-offs in discoverability, caching, and tooling. They are not interchangeable defaults; pick based on what has to route and cache the request.
- Deprecating an API version safely means a time-bound, instrumented lifecycle — not documentation alone. "It says deprecated in the docs" is not an operational control.

---

## The Backward Compatibility Boundary

**What it is:** The strict classification of any API change as backward compatible (safe to deploy immediately) or breaking (unsafe without a version boundary).

**Why it exists:** Engineers routinely ship breaking changes believing them to be minor — adding a validation rule, renaming a badly chosen field. To a downstream consumer, an unannounced breaking change is indistinguishable from a server crash.

**Options:**
1. **Backward compatible** — adding optional fields, adding new endpoints, relaxing constraints, accepting a wider range of valid input
2. **Breaking** — removing fields, changing a field's type, adding new *required* input, tightening constraints

**Trade-offs:**
- *Staying strictly compatible:* safe to deploy at any time with no consumer coordination, but the surface can only grow — deprecated fields and legacy parameters accumulate indefinitely as structural tech debt.
- *Allowing breaking changes:* cleans up that debt and aligns the contract with the current domain model, but breaks every consumer who hasn't rewritten their integration to match.

**When to choose each:**
- *Compatible changes:* the default for essentially all routine feature work — bias toward adding a new optional field over modifying an existing one.
- *Breaking changes:* only as part of a planned major version, with the business explicitly accepting the cost of a deprecation and migration cycle.

**Common failure modes:**
- **The tightened-validation trap:** rejecting ages over 150 as biologically implausible looks like a bug fix. A legacy automated client that's been sending `199` due to its own bad data starts receiving HTTP 400 instead of 200 — an unannounced production outage caused by a change that looked purely defensive.
- Treating field removal as equivalent to deprecation, or assuming "I only added an optional field" is automatically safe when a consumer relies on its *absence*.

**Example:** Google's API Improvement Proposals (AIP-206) define the boundary precisely: adding a response field is never breaking; changing a field's type is *always* breaking, even widening `int32` to `int64`, because a statically typed client will fail to deserialize the new payload regardless of whether the new type is technically a superset. **[Consensus: classify every API change as compatible or breaking before shipping it — "probably fine" is not a classification]**

---

## API Versioning Strategies

**What it is:** The routing and parsing mechanism that lets a provider run multiple, incompatible API contracts simultaneously while consumers migrate between them.

**Why it exists:** Breaking changes are eventually unavoidable. Once one ships, the old and new contracts have to be served concurrently, and something has to route each request to the right implementation.

**Options:**
1. **URI versioning** — the version is encoded in the path (`/v1/users`, `/v2/users`)
2. **Header-based versioning** — the version travels in a header (`Accept: application/vnd.api+json;version=2`)
3. **Schema-level versioning** — compatibility is encoded directly into a binary serialization format (Protobuf, Avro) via field tags, independent of any HTTP-level scheme

**Trade-offs:**

| Strategy | Buys | Costs |
|---|---|---|
| URI versioning | Trivial to route at a load balancer, highly discoverable, caches cleanly with standard CDN rules | Technically misuses REST semantics (a version isn't really a distinct resource); duplicated routes internally |
| Header-based versioning | Keeps the URL as pure resource identity | Breaks naive HTTP caching (needs `Vary: Accept`), harder to debug with a plain `curl`, requires header inspection at the gateway |
| Schema-level versioning | Compatibility enforced at compile/serialization time, not by convention | Requires the whole ecosystem to adopt a schema registry and give up human-readable payloads |

**When to choose each:**
- *URI versioning:* public web APIs and external integrations — the default.
- *Schema-level versioning:* high-throughput internal service-to-service RPC.
- *Header-based versioning:* rarely the right default; mainly seen where a legacy enterprise REST standard mandates it.

**Common failure modes:**
- **The minor-version route:** using `/v1.1/`, `/v1.2/` for changes that were already backward compatible, forcing consumers to update and redeploy just to receive additions that shouldn't have required any client change at all — defeating the entire point of compatibility.
- **Silent field-number reuse:** in a schema-level scheme, deleting a field and later reassigning its integer tag to something new. Old clients deserialize the new field using the old field's meaning, silently — no error, just wrong data.

**Example:** Protobuf's field numbering is the canonical case of schema-level versioning: fields are identified by integer tag (`string name = 1;`), not name, so renaming is safe and new fields can be added freely — but a deleted field's tag must never be reused, or old clients will misinterpret new data as the old field. The compatibility rule is enforced by the schema mechanics themselves, not by a human remembering a convention. **[Strong Recommendation: URI versioning by default for anything externally consumed — operational simplicity at the load balancer beats theoretical REST purity]**

---

## The Sunset Pattern

**What it is:** A structured, time-bound lifecycle for retiring an obsolete API version: active → deprecated (with warnings) → sunset (active disruption) → removal.

**Why it exists:** Supporting every past version forever is its own form of unmanageable complexity. Consumers eventually have to migrate — but predictably, not by surprise.

**Options:**
1. **Permanent support** — the old version lives indefinitely, mapped onto the current backend through accumulating translation layers
2. **The sunset pattern** — a deprecation window with escalating, programmatic signals, ending in scheduled removal

**Trade-offs:**
- *Permanent support:* zero consumer friction — a decade-old integration keeps working — but the provider's codebase becomes a permanent museum of legacy adapters, an indefinite operational tax.
- *Sunset pattern:* keeps the codebase clean and the attack surface small, but pushes real, uncompensated migration work onto consumers; too short a window and consumers abandon the platform instead of migrating.

**When to choose each:**
- *Permanent support:* infrastructure with an enormous, uncoordinatable installed base — hardware drivers, OS syscalls, foundational wire protocols.
- *Sunset pattern:* the default for SaaS APIs, SDKs, and internal services.

**Common failure modes:**
- **The silent deprecation:** documentation says a version is "deprecated," but nothing programmatic enforces it. Years later the provider finally shuts off the old servers and discovers they were still carrying 40% of production traffic. Documentation is not an operational control.

**Example:** The PostgreSQL wire protocol takes the permanent-support path — its binary protocol has stayed stable across decades of major versions, so a driver written in 2005 still works today. A typical SaaS sunset instead uses the IETF `Deprecation` and `Sunset` HTTP headers on a real timeline: a year out, responses carry `Deprecation: true`; six months out, a `Sunset: <date>` header appears; a month out, the provider runs brief scheduled "brownouts" — deliberate 503s for a few minutes at a time — specifically to trip the alerting systems of legacy consumers who haven't been watching the headers.

---

## Why Smart Engineers Disagree: URI vs. Header Versioning

The classic fight in API versioning is architectural purity against operational pragmatism.

REST purists argue that a URL identifies a resource, not a representation of it — a `User` is the same resource whether the response is shaped like v1 or v2. By that logic, URI versioning (`/v1/users`) is a category error, and the version belongs in a content-negotiation header instead.

Operationally minded engineers — SREs, infrastructure teams — push back hard. Caching on header values is fragile, routing on headers requires deeper inspection at every load balancer, and a header isn't something you can paste into a Slack message or grep out of an access log when a customer files a support ticket at 2 a.m.

The pragmatic case wins in practice: REST's theoretical purity buys nothing if the API can't be cached by a CDN, routed by an ordinary Layer 7 load balancer, or debugged quickly by whoever is on call. URI versioning is the dominant real-world choice precisely because it aligns with how web infrastructure actually works, trading a small amount of semantic purity for a large amount of operational resilience.

*Concepts expanded in later chapters: REST vs. RPC transport mechanics (Part III, Ch 19); branching and release strategy for the code that implements these versions (Part VII, Ch 50, Ch 56).*
