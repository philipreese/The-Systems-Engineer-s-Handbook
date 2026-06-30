# Chapter 20 — Resource Modeling

**Prerequisites:** [Part II, Ch 04 — Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Ch 15 — API Surface Design: What to Expose, What to Hide](../part02-software-architecture/ch15-api-surface-design-expose-hide.md), [Part III, Ch 19 — REST vs. RPC vs. Event-Driven](ch19-rest-vs-rpc-vs-event-driven.md). Specifically: information hiding, minimal surface area, progressive disclosure, and REST's resource-oriented ontology.

**New vocabulary introduced:** the action problem

**Key takeaways:**
- Ch 19 decided to expose resources. This chapter is about what a resource actually is in your domain: a stable, externally meaningful noun with identity and state — not a database table, not an ORM model, and not a controller method wearing a URL.
- Model resources around the consumer's domain language, not the provider's storage layout. A resource that mirrors the database schema couples every client to internal refactors the client should never see.
- Nest URIs shallowly — about one level deep is the practical default. Every additional level of hierarchy encodes a traversal dependency, not just a relationship, and it's the traversal dependency that breaks later.
- Real systems have operations that don't map to CRUD. The sub-resource action endpoint (`POST /orders/{id}/cancel`) is the durable answer: it keeps the resource as the noun while giving complex state transitions an explicit, unambiguous trigger — generic command endpoints quietly turn REST back into RPC with extra steps.

---

## Domain Nouns, Not Schema Projections

**What it is:** Defining URIs and resource payloads around business concepts the consumer understands, deliberately hiding the provider's internal database schema rather than reflecting it.

**Why it exists:** A resource that's just a serialized database table couples the API contract to the provider's storage engine. If the provider normalizes a table, the contract breaks — and the consumer was never supposed to know the table existed in the first place.

**Options:**
1. **Schema-driven resources** — URIs map 1:1 onto internal tables (`/users`, `/user_roles`, `/user_preferences` as separate endpoints)
2. **Domain-driven resources** — URIs map onto aggregated business concepts (a single `/accounts` resource, internally assembled from several normalized tables)

**Trade-offs:**
- *Schema-driven:* fastest to build — an ORM entity serializes straight to JSON with no mapping code — but it destroys information hiding outright, forcing the consumer to understand the provider's relational structure and often to make several calls to reassemble one business entity.
- *Domain-driven:* isolates the external contract from internal storage volatility entirely, so the database can be refactored without breaking clients — but it costs a real translation layer (DTOs, Ch 11) and runtime aggregation overhead.

**When to choose each:**
- *Domain-driven:* the default for any external, public-facing API or cross-service contract.
- *Schema-driven:* acceptable only for rapid internal tooling where the consumer and the database are owned by the exact same team.

**Common failure modes:**
- **The normalization leak:** an engineer normalizes `shipping_address` out of the `orders` table for storage reasons. Because the API was schema-driven, `/orders` silently stops returning the address, breaking every client that expected it — a storage optimization became an uncoordinated breaking change.
- Exposing raw table names as resources (`/user_table`, `/order_rows`), so the resource shape changes every time the schema does.

**Example:** Stripe's `Charge` object is a single, cohesive domain noun. Fetching one returns one clean JSON entity, even though constructing it internally means querying legacy banking gateways, fraud models, and a dozen heavily sharded tables. The client never has to know any of that exists. **[Strong Recommendation: model the resource the consumer needs, then build the translation layer to produce it — never the reverse]**

---

## Hierarchical Nesting Limits

**What it is:** The constraint applied to URI depth when one resource logically belongs to another (an item within an order within a user).

**Why it exists:** A resource's identity should stand on its own. Every level of nesting beyond what's needed for scoping ties that resource's identity to one specific traversal path through its parents — a structural dependency disguised as a URL convention.

**Options:**
1. **Deep nesting** — the URI reflects the full ownership chain (`/users/{user_id}/orders/{order_id}/items/{item_id}`)
2. **Shallow nesting (≈1–2 levels)** — nesting is used only for collection scoping; individual resources get flat, globally unique identifiers (`/users/{user_id}/orders` to list, but `/orders/{order_id}` and `/items/{item_id}` to fetch directly)

**Trade-offs:**
- *Deep nesting:* gives absolute contextual clarity in the URL itself — but it's brittle. If an item is ever allowed to belong to more than one order, the URL structure has no way to represent that, because it never expected to.
- *Shallow nesting:* maximizes routing flexibility and gives every resource a stable identifier that survives changes to the business logic above it — but it requires globally unique IDs (not composite keys scoped to a parent) and pushes authorization out of the URL path and explicitly into the business tier, where it has to be handled on purpose.

**When to choose each:**
- *Shallow nesting:* the default. Stop nesting after one level.
- *Deep nesting:* only when a child resource is mathematically incapable of existing without its parent and relies on a sequential ID scoped to that parent (`/orders/123/lines/1`).

**Common failure modes:**
- **The traversal trap:** a client receives an alert containing an `item_id`. Because the API enforces deep nesting, the client can't just fetch the item — it has to brute-force search the parent `/orders` endpoints first, just to reconstruct a URL it should have been able to build directly.
- URLs that mirror an ORM join path rather than a relationship a consumer actually needs (`/orgs/{id}/teams/{id}/members/{id}/permissions`), where every level the client wasn't already holding has to be discovered first.

**Example:** GitHub nests one level deep because an issue is genuinely scoped to a repository: `/repos/{owner}/{repo}/issues`. It stops there — fetching a specific comment doesn't require walking through the issue number first; once an entity has its own global identity, GitHub accesses it directly rather than continuing to nest.

---

## The Action Problem

**What it is:** The recurring case where a real business operation — canceling an order, refunding a payment, publishing a draft — doesn't map cleanly onto `PUT` (full replace) or `PATCH` (partial update).

**Why it exists:** HTTP verbs are infrastructural; business processes are domain-specific. Forcing a multi-step state transition into a generic CRUD verb hides the actual business intent from the request itself, and that hidden intent is exactly where dangerous side effects get lost.

**Options:**
1. **Generic state patching** — the client mutates a status field directly (`PATCH /orders/123 {"status": "canceled"}`)
2. **Sub-resource action endpoint** — the provider exposes an explicit verb on the resource (`POST /orders/123/cancel`)
3. **Shapeless command endpoint** — the provider abandons resource routing entirely (`POST /cancelOrder`)
4. **Separate action resource** — the action itself becomes a resource with its own lifecycle (`POST /refunds {"charge_id": "ch_1"}`), reserved for actions complex enough to deserve one

**Trade-offs:**
- *Generic state patching:* architecturally the "purest" REST, but the server has to diff the incoming payload against current state to even infer what the client meant — and if that inference is wrong, a side effect like an actual refund can silently never fire.
- *Sub-resource action:* keeps the resource as the noun while making the command explicit and unambiguous — at the cost of a non-noun path segment that offends strict REST purists.
- *Shapeless command endpoint:* maximizes clarity for that one operation, but quietly turns the whole API into RPC with extra steps, losing the predictability and cacheability the resource model was for.

**When to choose each:**
- *Sub-resource action:* the default for state-machine transitions and any mutation that triggers a real side effect.
- *Generic state patching:* literal data-entry updates only — changing a display name, not canceling an order.
- *Separate action resource:* when the action itself has a lifecycle worth tracking independently (a `Refund`, a `Dispute`).

**Common failure modes:**
- **The implicit side effect:** a client sends `PATCH /invoices/1 {"status": "paid"}` to update the UI. The database write succeeds, but because the explicit intent was never expressed, the integration that actually charges the payment gateway never fires. The invoice says paid; the money was never collected.
- Generic `/doSomething`-style endpoints accumulating over time as the path of least resistance for whatever didn't fit cleanly elsewhere.

**Example:** Stripe's `PaymentIntent` models a transaction's lifecycle as an explicit state machine. You don't `PATCH` a payment through security checks — you call `POST /v1/payment_intents/{id}/capture` or `.../cancel`. The resource stays the noun; the transitions are explicit sub-resource verbs, which is also POST's natural role here as the verb of last resort for anything that doesn't fit `GET`/`PUT`/`PATCH`/`DELETE`. **[Consensus: sub-resource action endpoints over generic command endpoints, the moment a mutation has a real side effect attached to it]**

---

## Resource Relationships: Embedding vs. Referencing

**What it is:** Whether a resource includes the full data of a related resource directly in its payload (embedding) or only a reference to it that requires a follow-up request (referencing).

**Why it exists:** This is a direct trade between network round-trips and payload size — and, less obviously, between developer convenience and cache correctness.

**Options:**
1. **Referencing** — the parent includes only an ID or URI of the related resource (`"user_id": "usr_123"`)
2. **Embedding** — the parent serializes the full related object directly into its own payload
3. **Hybrid / expansion** — referencing by default, with embedding available on request through a progressive-disclosure parameter (Ch 15) like `?expand=user`

**Trade-offs:**
- *Referencing:* keeps payloads minimal and cache boundaries clean — if the referenced resource changes, the parent's cached representation is still valid — but it forces the client into additional round-trips (the N+1 problem) to assemble the full picture.
- *Embedding:* one round-trip gets everything, which is a genuinely better experience for the client — but payloads bloat with data the client may not need, and cache invalidation becomes effectively impossible, since a change anywhere in the embedded graph invalidates the parent too.

**When to choose each:**
- *Referencing:* the default for cross-domain entities and any public API.
- *Embedding:* only for a child resource that's structurally bounded by its parent with no standalone lifecycle (line items inside an invoice).
- *Hybrid:* most production APIs in practice — reference by default, let the client opt into the round-trip cost it actually wants.

**Common failure modes:**
- **The accidental god object:** an engineer embeds relations by default to optimize for the UI. Fetching a `Comment` embeds its `Author`, which embeds the author's `Company`, which embeds the company's `BillingHistory`. A request for a ten-word comment generates a multi-megabyte payload and exhausts the database connection pool — entirely as a side effect of how deeply embedding was allowed to cascade.

**Example:** GitHub leans on referencing to protect its own infrastructure: fetching a pull request returns flat IDs and explicit hypermedia links (`"commits_url": "https://api.github.com/..."`), and the client has to traverse the link deliberately rather than receiving an unbounded object graph by default.

---

## Why Smart Engineers Disagree: Purity vs. Pragmatism in Action Verbs

The most persistent fight in resource modeling is the action problem, and it's a disagreement about what REST's noun constraint is actually for.

REST purists argue that everything can and must be a noun. Canceling an order should never be `POST /orders/123/cancel` — it should be `POST /cancellations {"order_id": 123}`, creating a `Cancellation` as an independent, trackable resource. This preserves the strict noun/verb uniformity REST promises, and it has a real benefit: the cancellation itself now has an identity, a history, and can be queried later.

Pragmatic engineers see this as architectural gymnastics. A cancellation is an action, not an entity, and forcing it to masquerade as one creates a confusing developer experience — the client now has to manage the lifecycle of an object that exists purely to satisfy a modeling rule, not because the business actually thinks of "a cancellation" as a thing with its own identity.

The deciding question isn't which side is more correct about REST theory — it's whether the resource is stable and independently meaningful enough to deserve being modeled as a noun in the first place. A `Refund` genuinely has its own lifecycle (it can be issued, reversed, disputed) and earns the separate-resource treatment. A plain order cancellation usually doesn't — and treating it as a manufactured noun anyway is accidental complexity (Ch 02) bought for theoretical purity rather than for anything the consumer needed. Optimize the boundary for the human reading the API, not for uniformity with no practical payoff.

*Concepts expanded in later chapters: error response structure for these endpoints (Part III, Ch 21), idempotency guarantees for state-mutating actions (Part III, Ch 22), pagination of resource collections (Part III, Ch 23).*
