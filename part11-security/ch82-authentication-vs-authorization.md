# Ch 82 — Authentication vs. Authorization

**Prerequisites:** [Authentication and Authorization Boundaries](../part03-api-design/ch24-authentication-authorization-boundaries.md) (confused deputy problem, zero-trust architecture, where authorization logic architecturally lives), [Defense in Depth](ch80-defense-in-depth.md) (independent, orthogonal controls), [Input Validation](ch81-input-validation.md)

**New vocabulary introduced:** RBAC, ABAC, delegated authorization

**Key takeaways:**
- Authentication verifies *who* a caller is; authorization decides *what that verified caller is allowed to do*. Ch 24 already drew this line — a system can authenticate a caller flawlessly and still authorize the wrong thing, or vice versa. This chapter goes deep on the mechanics of each, not where the check runs, which Ch 24 already settled.
- [Legitimate Trade-off] A server-held session can be revoked instantly because the server holds the authoritative state; a signed token (a JWT) needs no server-side lookup to verify but can't be revoked before its expiry without adding back the same stateful check tokens exist to avoid. Neither dominates — the choice depends on how much a specific system needs instant revocation versus lookup-free scaling.
- [Consensus] OAuth 2.0 and OpenID Connect are the dominant real-world standard for delegated authorization and federated identity — letting one service act on a user's behalf, or a user authenticate through a third-party identity provider, without that service or provider ever learning the user's primary credential.
- [Strong Recommendation] RBAC is simpler to reason about and audit when permissions map onto a small number of stable roles; ABAC scales better once access depends on runtime attributes (resource ownership, time, location) a fixed role hierarchy can't express — at a real cost to auditability that RBAC doesn't pay.
- A verified identity and a correct permission decision are independent guarantees. The 2018 Equifax breach turned a single authenticated internal service into 147 million exposed records precisely because nothing behind the authentication check verified what that authenticated identity was actually authorized to query.

---

Authentication and authorization get conflated in casual engineering conversation the same way Ch 76 warned concurrency and parallelism get conflated: two structurally separate questions, routinely discussed as though answering one answers the other. Authentication verifies identity — *who is making this request*. Authorization evaluates permission — *what is this now-verified caller allowed to do*. Ch 24 already established that distinction and settled where the authorization check architecturally belongs: in the owning service, not stripped out at the edge. This chapter does not relitigate that placement. It goes deep on the mechanics Ch 24 deferred by number — how identity actually gets verified, and how a permission decision actually gets made, wherever in the system that logic runs.

Neither question implies the other. A system can authenticate a caller with cryptographic certainty and still authorize far too much once that identity is confirmed. A perfectly designed permission policy is equally meaningless if the system mistakes an attacker for the legitimate caller in the first place. Treating "the user is logged in" as sufficient justification for an action is the single most common instance of this confusion, and it is a category error, not a shortcut — logged in answers only the first question.

### Decision: Authenticate With Server-Held Sessions, or Self-Contained Signed Tokens

**What it is:** Where authenticated identity state lives once a caller has proven who they are. A session stores the authoritative record server-side and gives the client an opaque identifier — typically a cookie — to present on each request. A token, most commonly a signed JWT, carries the identity claims themselves, verified by checking a cryptographic signature rather than by looking anything up.

**Why it exists:** Both approaches answer the same question, but the trade-off is unavoidable: authoritative state either lives on the server, where it can be changed the instant it needs to be, or it's distributed to the client, where verification is fast and requires no shared lookup but nothing can be un-issued once it's out. A signed token's integrity guarantee — nobody can forge or alter it without the signing key — is not the same as a confidentiality guarantee; unless separately encrypted, anyone holding a JWT can typically read every claim inside it.

**Options:**
- **Server-side sessions** — an opaque session ID; every request resolves it against a shared, authoritative store (a database, a Redis cache).
- **Signed tokens (JWTs)** — a self-contained, cryptographically signed document; every service verifies it locally against a public key, with no shared store consulted per request.

**Trade-offs:** Sessions give immediate, deterministic revocation — compromise an account and the server invalidates it centrally, effective on the next request — at the cost of a stateful lookup on every request and a shared store that becomes both a scaling bottleneck and a single point of failure under load. Tokens eliminate that lookup entirely, letting any service verify identity from local memory and a cached public key, which scales cleanly across distributed services — but revocation before expiry isn't naturally possible at all; a stolen token stays valid for its full lifetime unless the system adds back an out-of-band revocation check, which reintroduces the exact stateful lookup tokens exist to avoid.

**When to choose each:** [Legitimate Trade-off] Sessions where immediate revocation is a real requirement — an administrative console, a banking back office, anywhere an active compromise has to be shut off the instant it's detected, not five minutes later. Tokens where services must verify identity independently at scale without a shared store on the hot path — distributed microservices, high-throughput public APIs. A workable middle ground scopes tokens by risk: short-lived, purely stateless tokens for low-risk read traffic, paired with a state-checked gate in front of high-value write or administrative operations, so the parts of the system that most need instant revocation are the parts that pay the lookup cost for it.

**Common failure modes:** A token issued with a long expiry and no revocation path at all, so a "logout" button that only deletes the client-side copy leaves the token itself fully valid and usable by anyone who intercepted it beforehand; treating a signed token as encrypted and storing sensitive data inside its claims; a symmetric signing secret that's weak or accidentally checked into a repository, letting anyone who finds it forge tokens for any identity the system trusts.

**Example:** Most traditional web applications authenticate through server-managed sessions with a cookie. Modern token-based APIs commonly issue JWTs verified independently by every downstream service against a JSON Web Key Set (JWKS) endpoint published by the issuer — each service caches the public key according to standard HTTP cache headers and verifies signatures with zero network round-trip per request.

### Decision: Delegate Identity Through OAuth 2.0 / OIDC, or Share Primary Credentials Directly

**What it is:** Whether a service acting on a user's behalf, or a user authenticating into a third-party application, does so by sharing that user's actual primary credential, or by obtaining a narrowly scoped, time-limited credential issued by a dedicated identity provider that never exposes the primary one. This is where Ch 24's deferred "cryptographic mechanics of authentication protocols" resolves.

**Why it exists:** An application that needs limited access to a user's data on another service has no reason to ever see that user's password — doing so multiplies the number of places a single credential can leak, and gives the borrowing application far more access than the specific task requires. OAuth 2.0 solves the delegation problem: a user authenticates once with an identity provider, which issues a scoped credential the requesting application uses instead of the user's actual password. OpenID Connect (OIDC) is a thin, standardized identity layer on top of OAuth 2.0, adding a verifiable "who is this" claim to what OAuth alone only handles as "what can this act on."

**Options:**
- **Direct credential sharing** — a user's primary password is entered into, and potentially stored by, every application that needs access.
- **OAuth 2.0 delegated authorization** — a third party receives a scoped, revocable credential without ever learning the primary one.
- **OpenID Connect federated authentication** — OAuth 2.0 extended with a standardized identity assertion, letting a user authenticate into an application through an external identity provider.

**Trade-offs:** Delegation eliminates password sharing across services and lets access be scoped and revoked independently of the primary credential, at the cost of genuine protocol complexity — multiple interacting components (authorization server, resource server, redirect flows) that have to be implemented correctly, with a well-documented history of subtle misconfigurations. Direct credential sharing has no protocol overhead at all, but every application holding the primary credential is now a place that credential can leak from, and revoking access to one application means rotating a password used everywhere else too.

**When to choose each:** [Consensus] OAuth 2.0 whenever one service needs to act on a user's behalf without learning their primary credential — which is effectively always, once more than one party is involved. OIDC on top of it whenever the relying application also needs standardized, verifiable identity information about the user, beyond delegated access to a resource alone. Direct credential sharing has no legitimate case in a multi-party system; it survives mainly in legacy integrations that predate a federated alternative.

**Common failure modes:** A misconfigured redirect URI that lets an attacker intercept an authorization code meant for the legitimate application; scopes requested far broader than the task needs, so a compromised integration inherits access well past what it was supposed to have; access tokens issued with lifetimes long enough that a leak has a wide window of usefulness; a relying application that trusts an identity provider it never properly validated the trust relationship with.

**Example:** Identity providers including Okta, Microsoft Entra ID, and Keycloak implement OAuth 2.0 and OIDC as their primary federated-identity architecture: a user authenticates once with the provider, which issues a signed access token that downstream services verify against the provider's published JWKS endpoint, and refresh tokens rotate at the edge to keep access-token lifetimes short without forcing the user to re-authenticate constantly.

### Decision: Model Permissions With RBAC, or ABAC

**What it is:** Once identity is established, how the system decides whether the requested action is permitted. **RBAC** (Role-Based Access Control) groups permissions into named roles and assigns callers to roles. **ABAC** (Attribute-Based Access Control) evaluates a policy against runtime attributes of the subject, resource, action, and environment at the moment of the request, rather than a fixed role membership.

**Why it exists:** Organizations express permission differently depending on how stable their structure is. Where roles map cleanly onto real, durable job functions, a small number of named roles captures the permission model completely and stays auditable as the organization changes. Where access genuinely depends on context — who owns this specific resource, what time it is, which project this request is scoped to — a fixed role can't express the condition without either exploding into a role per combination or losing the distinction entirely.

**Options:**
- **RBAC** — permissions bundled into roles (`BillingAdministrator`, `SupportEngineer`); a caller's access is the union of their assigned roles' permissions.
- **ABAC** — policies evaluated against subject, resource, action, and environment attributes at request time, with no fixed role as an intermediate concept.

**Trade-offs:** RBAC is intuitive to reason about, fast to evaluate (a set-membership check), and straightforward to audit — "what can this role do" is a query anyone can answer by reading the role definition. It cannot natively express a relational constraint like "only if the requester and the resource owner share a department," and forcing that kind of context into RBAC tends to produce role explosion: a new role for every combination of title, region, and constraint, until the role list itself becomes unauditable. ABAC expresses exactly those relational and contextual constraints natively and adapts to runtime conditions RBAC has no vocabulary for, at the cost of a policy surface that's fundamentally harder to audit — because access is computed dynamically from an open-ended combination of attributes, "what can this user access" often has no static answer at all, only an answer computed per request.

**When to choose each:** [Strong Recommendation] RBAC when permissions map onto a genuinely small, stable set of organizational roles and auditability matters as much as expressiveness. ABAC when access legitimately depends on combinations of resource ownership, time, location, or other runtime context that a role hierarchy would have to fake by proliferating roles. Hybrid systems are common and often correct: RBAC for coarse-grained access, ABAC layered on top for the finer-grained constraints RBAC structurally can't express.

**Common failure modes:** *Role explosion.* A team forces regional data isolation into RBAC by creating a role per title-department-region combination — `Manager-NorthAmerica-Logistics`, `Manager-AsiaPacific-Finance` — until the system accumulates thousands of roles nobody can audit with confidence, the exact failure the role model was chosen to avoid. On the ABAC side: policies complex enough that no one can state with confidence what a given user can currently access, because the answer depends on live external attribute lookups evaluated fresh on every request.

**Example:** Kubernetes implements RBAC directly: a `Role` binds a set of verbs (`get`, `list`, `update`) to a resource type within a namespace, and a `RoleBinding` attaches that role to a user, group, or service account. The model stays auditable at cluster scale because permissions are grouped into a comparatively small number of stable roles rather than computed per request.

### Why Smart Engineers Disagree on Real-Time Revocation

Authentication mechanisms mostly converge on industry standards; how aggressively to guard against a leaked token in flight does not.

One position treats statelessness as close to a hard constraint: the entire point of moving off session cookies was eliminating the shared-store bottleneck, and reintroducing a real-time revocation check — querying a shared cache on every request to see if a token has been blacklisted — quietly reverses that decision while keeping the added complexity of tokens. This position manages the residual risk with short token lifetimes and frequent refresh-token rotation at the edge instead of a lookup in the hot path.

The opposing position treats a short expiry as an insufficient control on its own: during an active credential leak, even a five-minute window of unrevocable access to sensitive systems is real exposure, and relying on expiry alone ignores Ch 80's containment argument — an assume-breach posture doesn't get to assume the breach will be short. This position accepts a sub-millisecond revocation-list check as the cost of actually bounding an active compromise rather than waiting it out.

Both are optimizing the same trade-off from different sides: I/O performance against how long an active compromise stays live. The resolution in practice is rarely all-or-nothing — risk-weighted token lifecycles, where low-risk read traffic gets fully stateless short-lived tokens and high-value write or administrative operations get a state-checked gate, spend the lookup cost only where an uncontained compromise would actually be expensive.

### Case Study: The 2018 Equifax Breach

The 2018 Equifax breach — 147 million exposed records — is the clearest illustration available of why flawless authentication and correct authorization are independent guarantees, not one implying the other.

The initial foothold came from an unpatched remote-code-execution flaw in Apache Struts at the external perimeter, letting the attacker reach an internal dispute-portal application. From there, the failure that turned one compromised service into a company-wide breach had nothing to do with authentication: the dispute portal held entirely legitimate, correctly authenticated credentials to query backend data stores. Internal databases verified that authentication token, confirmed the dispute service's identity, and stopped checking.

Nothing behind that authentication check verified *what* the dispute service should have been authorized to query. There was no scoped policy — RBAC or ABAC — constraining the authenticated identity to records actually linked to an active dispute case. With authentication satisfied and authorization absent, the attacker used the dispute service as a confused deputy (Ch 24) and ran broad, unrestricted queries across unrelated customer tables for months, undetected.

Had an authorization gate enforced that the dispute service could only reach records tied to an active, audited case — a narrow RBAC role or an ABAC policy keyed to case ownership — the compromised perimeter service would have hit a wall immediately after authenticating. The identity check worked exactly as designed. It was never the control that was supposed to stop this.
