# Chapter 24 — Authentication and Authorization Boundaries

**Prerequisites:** [Part I, Ch 06 — Cost Models and Mechanical Sympathy](../part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md), [Part II, Ch 13 — Coupling and Cohesion at the Architecture Level](../part02-software-architecture/ch13-coupling-cohesion-architecture-level.md), [Ch 15 — API Surface Design: What to Expose, What to Hide](../part02-software-architecture/ch15-api-surface-design-expose-hide.md), [Part III, Ch 21 — Error Handling Contracts](ch21-error-handling-contracts.md). Specifically: the network latency tax, bounded contexts, and minimal API surface area.

**New vocabulary introduced:** confused deputy problem, zero-trust architecture

**Key takeaways:**
- Every request has to answer two separate questions: who is this caller (authentication), and what are they allowed to do (authorization). This chapter is about where those two checks get enforced architecturally — not the cryptography behind either one.
- Authentication is cheap to centralize at a gateway because identity is the same fact everywhere. Authorization can't be — only the owning service has the domain data to know whether *this* user can act on *this* resource, which means fine-grained authorization has to live in the service, not the edge.
- A token propagated downstream after the initial authentication is what lets internal services skip re-authenticating a user on every hop — but propagating it blindly, without verifying it at each boundary, is exactly how the confused-deputy problem happens: a trusted internal service gets tricked into using its own elevated privilege on a caller's behalf without checking whether that caller was actually allowed to ask for it.
- Zero trust — verifying identity at every hop instead of trusting the network perimeter once at the edge — is the structural response to confused deputies, not an optional hardening step. The alternative is a system that's only as secure as its least-careful internal service.

---

## Enforcement Placement: Gateway vs. Service

**What it is:** Where the logic that verifies identity and evaluates permissions actually executes — once at the edge, independently in every service, or some combination of both.

**Why it exists:** This is a trade between centralized operational control and the kind of defense-in-depth that survives a single compromised component. Deciding whether to check identity at the front door or again at the destination changes how vulnerable the system is to lateral movement once something inside has gone wrong.

**Options:**
1. **Centralized gateway** — a single edge proxy authenticates the external request and passes a trusted identity header downstream
2. **Decentralized service enforcement** — every service independently verifies the raw identity payload (a token signature, an mTLS certificate) before acting on it

**Trade-offs:**
- *Centralized gateway:* maximizes velocity — services never implement auth middleware, and identity is checked once at the cost of a single CPU cycle at the edge — but it creates an implicit trust boundary. If the gateway is bypassed or any internal service is compromised, the rest of the backend has nothing else protecting it, because everything behind the gateway blindly trusts internal traffic.
- *Decentralized enforcement:* every hop independently verifies, so a single compromised service can't pivot to attack its neighbors — but it costs real CPU on every internal call and risks each team implementing the check slightly differently.

**When to choose each:**
- *Centralized gateway:* fine for authentication in a genuinely isolated internal network or a modular monolith — but never sufficient for authorization, since the gateway has no way to know whether a specific user can mutate a specific downstream resource.
- *Decentralized enforcement:* the default for authorization universally, and the default for authentication in any modern zero-trust microservice architecture.

**Common failure modes:**
- **The blind passthrough:** a gateway verifies an external token, strips it, and injects `X-User-Id: 123` for downstream services to read. A compromised internal service — or just a misconfigured one — makes a direct request to `billing-service` with `X-User-Id: 1` (the admin account). Because the service trusts the perimeter rather than verifying anything itself, the request executes with full admin authority, no security control anywhere in the path having actually checked it.

**Example:** Kong and Envoy are widely deployed as centralized gateways terminating external identity flows — but in modern Kubernetes deployments, Envoy is just as often run as a per-pod sidecar instead, which forces decentralized enforcement: mTLS and token signatures get verified at every service boundary, not just once at the edge. **[Strong Recommendation: authenticate at the edge for efficiency; authorize in the owning service, because only it has the domain context to make that call correctly]**

---

## Token Propagation and the Confused Deputy

**What it is:** How a caller's identity survives the trip from the gateway through however many internal services a request touches — and what happens when a service acts on a caller's behalf without actually verifying who that caller was.

**Why it exists:** If Service A calls Service B to fulfill a user's request, B needs to know who that user actually is to enforce any user-level authorization. Lose that identity across the hop, and B is left with only one option: trust A completely.

**Options:**
1. **Perimeter trust (service identity)** — Service A authenticates to Service B with its own service-level credentials; B trusts A's claim about what the user is allowed to do
2. **Delegated token forwarding (user identity)** — Service A forwards the original user's signed token intact; Service B verifies it independently

**Trade-offs:**
- *Perimeter trust:* simple, no token-forwarding plumbing required — but Service B has no actual visibility into who the real user is, and has no choice but to assume Service A already authorized whatever it's asking for.
- *Delegated forwarding:* preserves real user context all the way through the call chain, so authorization can be enforced correctly at the deepest layer that actually owns the decision — but it couples internal service calls to the external token format, and a service that logs or forwards the token to an untrusted third party leaks it.

**When to choose each:**
- *Perimeter trust:* acceptable only for genuinely non-interactive system jobs with no user context at all — a nightly batch process, a scheduled reconciliation job.
- *Delegated forwarding:* the default for any call chain that originated from an external user.

**Common failure modes:**
- **The confused deputy:** a user asks a report service to generate a summary. The report service queries a downstream user service using its own elevated, perimeter-trusted credentials rather than the user's actual token, intending to filter results locally. A bug in that filtering leaks another company's data into the response. The user service wasn't compromised — it was *confused*, tricked into trusting the report service's network identity instead of demanding the real caller's token.

**Example:** OAuth 2.0 and OpenID Connect are the dominant delegated-auth frameworks for exactly this reason: a client's signed JWT passes unchanged from the gateway through Service A to Service B, and B independently verifies the signature itself rather than trusting that A already checked it — structurally closing off the confused-deputy path before it can open. **[Consensus: forward and re-verify the caller's actual identity at every hop that makes an authorization decision; never substitute a service's own credentials for it]**

---

## Identity Representation: Reference vs. Value Tokens

**What it is:** Whether the credential passed over the wire is an opaque pointer requiring a lookup (a reference token) or a self-contained, signed payload a service can verify locally (a value token, typically a JWT).

**Why it exists:** This is the trade between immediate, centralized revocation and the latency cost of a network round trip to check every single request against a central identity provider.

**Options:**
1. **Reference tokens** — a meaningless random string (a session ID); every service calls the identity provider to validate it and fetch identity data
2. **Value tokens (JWTs)** — a signed JSON payload carrying identity and permissions directly; services verify the signature locally with no network call

**Trade-offs:**
- *Reference tokens:* revocation is immediate — ban a user, and the very next validation call fails — but every request now pays a synchronous call to a central identity provider, which becomes a global bottleneck at scale.
- *Value tokens:* eliminate that bottleneck entirely, verified locally in microseconds — but introduce the token-expiry problem: a banned user's already-issued JWT stays cryptographically valid across the whole internal network until it naturally expires, an unavoidable window where revocation simply doesn't apply yet.

**When to choose each:**
- *Reference tokens:* sensitive financial, healthcare, or administrative operations where instant revocation is a hard regulatory requirement, not a nice-to-have.
- *Value tokens:* the default for standard distributed REST APIs and service-mesh-internal traffic, where microsecond local verification matters more than instant revocation.

**Common failure modes:**
- **The unverifiable JWT:** a service decodes a JWT's base64 payload to read `user_id`, but skips the actual cryptographic signature check against the identity provider's public key. An attacker edits `user_id` directly, re-encodes it, and gains full access — because nothing ever confirmed the payload hadn't been tampered with.

**Example:** GitHub's personal access tokens are opaque reference tokens specifically so a compromised one can be killed instantly with a round trip to GitHub's servers. Kubernetes service account tokens, by contrast, lean on JWTs as value tokens, prioritizing fast, decentralized validation inside the cluster over instant revocation.

---

## Why Smart Engineers Disagree: Edge vs. Depth Authorization

The sharpest disagreement here isn't about authentication — almost everyone agrees that belongs at the edge. It's about where authorization should live.

Engineers optimizing for operational simplicity want the gateway to handle both: define "only admins can `POST /billing`" directly in the Envoy or Kong config, so security logic lives in one auditable place and unauthorized traffic never even reaches the internal network.

Engineers optimizing for domain cohesion and zero trust point out that the gateway fundamentally can't make most authorization decisions correctly. It might know a user has an "admin" role, but it has no way to know whether *this* user owns *this* specific invoice without querying the billing domain itself. Pushing authorization rules into gateway config also creates a strange temporal coupling: the platform team now has to update routing config every time the billing team adds a new domain rule that has nothing to do with routing.

The resolution isn't a 50/50 compromise — it's a split by what each layer actually has the information to decide. The gateway handles coarse authentication (is this signature valid, is this token expired) and the most superficial routing-level checks. Fine-grained authorization — can this specific user touch this specific resource — is mathematically required to live inside the owning service, because that service is the only place the domain data needed to answer the question actually exists. You secure the perimeter with identity. You secure the data with domain logic, and nowhere else.

*Concepts expanded in later chapters: cryptographic mechanics of authentication protocols, threat modeling, and secrets management (Part XI, Ch 79–84); RBAC/ABAC policy design in depth (Part XI, Ch 82).*
