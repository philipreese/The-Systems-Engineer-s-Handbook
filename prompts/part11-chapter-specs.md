# Part XI — Chapter Specifications

Pre-filled special instructions for each Part XI chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part XI: Part I (Ch 09's decision-frameworks chapter already used an
inconsistent-authentication example to illustrate a missing system-level decision, and its
blast-radius/reversibility framing is this Part's default lens for how much a given control is
worth: a low-reversibility, high-blast-radius exposure — a leaked signing key, a compromised
build pipeline — warrants far more deliberation than a low-stakes one; Ch 07's partial-failure and
fail-fast arguments underpin Ch 80's assume-breach framing directly); Part III (Ch 24 already
did substantial work this Part must not repeat: it defined the confused deputy problem and
zero-trust architecture, resolved where authorization logic should live — edge versus the owning
service — and explicitly deferred four things to this Part by number: cryptographic mechanics of
authentication protocols and threat modeling in general (Ch 79, Ch 82), RBAC/ABAC policy design in
depth (Ch 82), and secrets management (Ch 83)). Every chapter here should reference Ch 24's
confused deputy problem and zero-trust architecture as already-settled vocabulary, never redefine
them.

Several forward references from earlier parts must be honored here:
- Part III, Ch 24 deferred cryptographic authentication-protocol mechanics and RBAC/ABAC policy
  depth to this Part by number (Ch 82), and threat modeling and secrets management in general
  (Ch 79, Ch 83).
- Part VII, Ch 56 deferred the full supply-chain threat model that tag signing defends against to
  this Part by number (Ch 84) — that chapter established *how* to sign a release marker; this Part
  covers what signing is actually defending against.
- Part VII, Ch 61 deferred the full secrets-management treatment for publishing credentials —
  rotation, scoping, what happens when a credential leaks — to this Part by number (Ch 83), having
  already named OIDC-based trusted publishing as the direction of travel away from long-lived
  tokens.
- Part VII, Ch 63 deferred the security threat model of a compromised or malicious dependency to
  this Part by number (Ch 84), distinguishing it explicitly from whether a dependency's version is
  current, which remains Ch 63's own subject.
- Part VIII, Ch 68 deferred credential handling in a runbook to this Part by number (Ch 83): a
  runbook step should point to where a credential is obtained, never embed one directly.
- Part IX, Ch 69 deferred both the general claim that logging is part of a system's attack surface
  and the specific mechanics of keeping a secret out of a log line to this Part by number (Ch 83),
  anchoring the forward reference in the Log4Shell vulnerability (CVE-2021-44228) as a concrete
  case where unvalidated input reaching a logging call became a remote-code-execution path — that
  same example is this Part's anchor for Ch 81, not Ch 83, since the defect was a missing input
  validation step, not a logging or secrets failure.
- Part IX, Ch 72 flagged that propagating trace context across an untrusted boundary can leak
  internal topology to anyone able to read it, and left the question entirely to this Part. It
  resolves here in Ch 79, as a concrete instance of enumerating attack surface at a trust boundary,
  not as a tracing-mechanics topic.

Boundary with adjacent Parts: Part III owns where authorization logic architecturally lives (edge
versus owning service) and the definitions of the confused deputy problem and zero-trust
architecture; this Part goes deeper on authentication and authorization mechanics (Ch 82) without
relitigating that placement argument. Part VI owns spec-first development and code review as
general engineering process; Ch 79 draws the same catch-it-early argument (Principle 10) into
threat modeling specifically, referencing Ch 46 rather than re-deriving why earlier review is
cheaper. Part VII owns release automation, tag signing, and dependency/toolchain currency
mechanics; this Part owns the threat models those mechanisms defend against (Ch 83, Ch 84), not
the mechanics themselves. Part IX owns what to log and when to page a human; this Part owns what
must never appear in a log line or a trace payload in the first place. Part XII (Performance)
has no dependency on this Part and is not a prerequisite; if a chapter here is tempted to discuss
the runtime cost of a control (hashing rounds, TLS handshake overhead, validation latency), it
should name the cost honestly as a trade-off and stop there — profiling methodology belongs
entirely to Part XII, not to this Part.

This Part moves from mental model to specific controls to the risk of what you didn't write
yourself: Ch 79 establishes the shared vocabulary and methodology (assets, adversaries, trust
boundaries, attack surface) every later chapter assumes without re-deriving. Ch 80 is the
organizing architectural principle — no single control is trusted alone — that explains *why*
Ch 81 through Ch 84 exist as independent, redundant layers rather than one perimeter check. Ch 81
and Ch 82 cover the two controls applied at nearly every trust boundary: validating what crosses
it, and deciding who and what is allowed to cross it. Ch 83 narrows to the specific artifact most
often responsible for turning a contained incident into a full compromise — the credential itself.
Ch 84 closes the Part by widening the aperture past code the team wrote at all, to code and
tooling it merely depends on.

---

## Ch 79 — Threat Modeling

```
This chapter opens Part XI by establishing the shared vocabulary and methodology every later
chapter in this Part assumes: assets (what's worth protecting), adversaries (who benefits from
compromising it and what they're capable of), trust boundaries (where trust level changes as data
or control crosses from one component to another), and attack surface (everything reachable from
outside the current trust boundary). Treat this as the Part's spine the same way Ch 74 was for
Part X — later chapters assume this framing already exists rather than re-deriving it.

Reference Ch 24's confused deputy problem and zero-trust architecture directly as worked examples
of trust-boundary reasoning already established in this handbook — do not redefine either term,
only point back to them as instances of the general methodology this chapter formalizes.

Cover a concrete, structured methodology rather than treating "think like an attacker" as
sufficient on its own: STRIDE (Spoofing, Tampering, Repudiation, Information disclosure, Denial of
service, Elevation of privilege) as a named, widely used categorization for walking a system's
trust boundaries and asking what can go wrong at each one. Take a position: threat modeling is a
design-time exercise, not a post-incident retrospective — this is Principle 10 (the cost of
correcting a mistake grows with the distance between when it's made and when it's discovered)
applied to security specifically, and the same argument Ch 46 already made for spec-first
development in general; reference Ch 46 directly rather than re-deriving why catching a design
flaw before implementation is cheaper than after.

Resolve Ch 72's deferred forward reference here: a distributed trace context propagated across an
untrusted boundary is a concrete instance of attack surface — it can hand an external party
internal service topology, naming conventions, or infrastructure details never intended to be
exposed. Use it as a worked example of enumerating what crosses a trust boundary, not as an excuse
to re-explain tracing mechanics.

Anchor in a well-documented, widely taught real incident where a narrow, seemingly low-value trust
boundary (a third-party vendor credential, an exposed internal service) turned out to be the actual
path to a much larger compromise — the kind of case study standard in security engineering
curricula — to make the trust-boundary argument concrete rather than abstract.

Do NOT cover: specific mitigations for any category STRIDE surfaces — input validation (Ch 81),
authentication/authorization mechanics (Ch 82), secrets handling (Ch 83), and dependency risk
(Ch 84) are each a later chapter's full subject, not this chapter's; re-deriving where authorization
logic should architecturally live (Ch 24, already settled); cryptographic protocol mechanics
(Ch 82).
```

---

## Ch 80 — Defense in Depth

```
This chapter is the organizing architectural principle for the rest of the Part: no single control
should ever be the only thing standing between an asset and a successful attack. It explains *why*
Ch 81 through Ch 84 exist as independent, redundant layers rather than a single perimeter check,
the same way Ch 74 explained the coordination-model spine the rest of Part X built on.

Take a position, anchored in this handbook's existing vocabulary rather than treated as a new
axiom: defense in depth is Ch 24's zero-trust architecture generalized past the single boundary
Ch 24 covered (service-to-service authorization) to every layer of a system — network, host,
runtime, application, data — and it is Ch 07's partial-failure and fail-fast arguments applied to
security specifically: assume any individual control will eventually fail or be bypassed, and
design so that failure is contained rather than catastrophic. Connect this directly to Ch 09's
blast-radius and reversibility framing: a layered set of independent controls is exactly how blast
radius is kept small when one layer is breached, and the number of layers a given asset warrants
should scale with how severe and irreversible its compromise would be, not be applied uniformly.

Cover the failure mode of perimeter-only security precisely: a single strong boundary control
(a firewall, a login page) that, once bypassed, grants an attacker unrestricted lateral movement
because nothing inside the perimeter re-verifies anything. Contrast this with layered,
independently-failing controls, where compromising one layer still leaves the asset protected by
the next.

Anchor in a real, widely documented breach where a narrow initial compromise (a third-party
vendor's credentials, a single exposed service) was able to move laterally through a flat internal
network with no additional internal controls — a canonical case study for exactly this argument,
already standard in security engineering literature — and contrast it with a real system's layered
control model (Kubernetes' combination of network policies, pod security standards, and RBAC as
independent layers that must each be bypassed in turn, none alone sufficient).

Do NOT cover: the mechanics of any single layer — input validation (Ch 81), authentication and
authorization (Ch 82), secrets management (Ch 83), or dependency trust (Ch 84) — each belongs
entirely to its own chapter; this chapter only establishes why having several, independent layers
is the correct default. Do not re-derive Ch 24's zero-trust definition or the confused deputy
problem; reference both directly.
```

---

## Ch 81 — Input Validation

```
This chapter covers the first concrete control most systems apply at a trust boundary: verifying
that data crossing into a more-trusted context is actually what it claims to be, before it's acted
on. It's the code-level instance of Ch 80's defense-in-depth argument — validate at every layer a
trust boundary is crossed, not only once at the system's outermost edge.

Resolve Ch 69's deferred forward reference in full here: the Log4Shell vulnerability
(CVE-2021-44228) is this chapter's anchor example, not a logging or secrets-management failure —
an unvalidated, attacker-controlled string (an HTTP header, for instance) reached a JNDI lookup
interpreter and triggered remote code execution. The defect was a missing validation step on
untrusted input before it reached an interpreter capable of acting on it; frame it that way
explicitly, distinct from Ch 83's ownership of what belongs in a log line at all.

Cover the general shape shared by an entire family of vulnerabilities — SQL injection, command
injection, template/expression-language injection (Log4Shell's category), cross-site scripting —
as the same underlying failure applied to different interpreters: untrusted input is concatenated
into a string that a downstream interpreter then executes or evaluates, rather than being kept
strictly as data. Take a position: parameterization (prepared statements, parameterized queries) is
the correct fix, not pattern-matching or blocklisting known-bad input — a blocklist only ever
covers attack patterns already known, while parameterization removes the interpreter's ability to
treat the input as anything other than data, structurally.

Cover allowlist validation (accept only known-good shapes: a specific format, a bounded length, a
constrained character set) as generally stronger than denylist validation (reject known-bad
patterns), and explain why: a denylist requires anticipating every malicious variant in advance,
while an allowlist only requires defining what's legitimately expected — a smaller, more stable
surface to get right.

Anchor in real systems: parameterized queries as implemented by virtually every mature database
driver (contrasted directly with string-concatenated SQL) as the canonical fix pattern; Log4Shell
itself as the flagship real-world case for why validating input before it reaches any interpreter —
not just a database — is the general principle, not a SQL-specific one.

Do NOT cover: authentication or authorization decisions (Ch 82); secrets or credential handling
(Ch 83); the shape of an error response returned to a caller after a validation failure, already
covered in Ch 21 — reference that chapter's error contract rather than re-deriving it.
```

---

## Ch 82 — Authentication vs. Authorization

```
This chapter goes deep on the mechanics Ch 24 explicitly deferred here by number: cryptographic
authentication-protocol mechanics and RBAC/ABAC policy design in depth. It assumes Ch 24 already
settled *where* authorization logic architecturally lives (the owning service, not the edge) and
does not relitigate that placement argument — this chapter is about the mechanics of authentication
and authorization themselves, wherever they're enforced.

Open by distinguishing the two terms precisely, since they're routinely conflated in casual
engineering conversation the same way concurrency and parallelism were in Ch 76: authentication is
verifying who a caller is; authorization is deciding what that verified caller is allowed to do.
A system can authenticate a caller perfectly and still get authorization wrong, and vice versa —
treat this as the chapter's first, foundational clarification.

Cover session-based authentication (a server-held session identifier, typically a cookie) against
token-based authentication (a self-contained, cryptographically signed token such as a JWT) as the
two dominant mechanisms, and take a position on the real trade-off: a session can be revoked
instantly server-side because the server holds the authoritative state; a signed token can't be
revoked before its expiry without an additional server-side revocation list, but avoids a
server-side state lookup on every request. Cover OAuth2/OIDC as the dominant real-world standard
for delegated authorization and federated identity (letting one service act on a user's behalf
without ever seeing that user's primary credential) — this is where Ch 24's deferred "cryptographic
mechanics of authentication protocols" resolves.

Cover RBAC (role-based access control) against ABAC (attribute-based access control) as the two
dominant authorization policy models, and take a position on when each fits: RBAC is simpler to
reason about and audit when permissions map cleanly onto a small number of stable roles; ABAC scales
better when access decisions depend on combinations of runtime attributes (resource owner, time of
day, request context) that a fixed role hierarchy can't cleanly express, at the cost of a
meaningfully harder-to-audit policy surface.

Anchor in real systems: OAuth2/OIDC as implemented by any major identity provider; JWTs as the
concrete token format underlying most modern token-based authentication; a real RBAC
implementation (Kubernetes' RBAC model, mapping subjects to roles to resource permissions) as a
concrete, widely encountered example.

Do NOT cover: where authorization logic should architecturally live — edge, gateway, or owning
service — already fully resolved in Ch 24; credential storage mechanics such as password hashing
or key management (Ch 83 — this chapter covers the protocol and policy decision, not how the
credential material backing it is stored or rotated); general input validation (Ch 81).
```

---

## Ch 83 — Secrets Management

```
This chapter resolves three explicit forward references made by number from earlier parts: Ch 61's
deferred full treatment of publishing-credential rotation and leak response, Ch 68's deferred rule
that a runbook step should point to where a credential is obtained rather than embed one, and
Ch 69's deferred claim that logging is part of a system's attack surface along with the specific
mechanics of keeping a secret out of a log line.

Define a secret precisely as the chapter's central vocabulary: any credential, key, or token whose
disclosure to an unintended party grants that party capability it shouldn't have — an API key, a
database password, a signing key, a session token. Cover why a secret checked into version control
or embedded in a config file committed alongside code is categorically worse than an ordinary bug:
version history is close to permanent, so a single commit can leak a credential that remains valid
long after the offending line is removed, unless the credential itself is also rotated.

Cover rotation and scoping as the two central mitigations: rotation limits how long a leaked
credential remains useful; scoping (issuing the narrowest-privilege credential that can do the
required job, and preferring short-lived, dynamically issued credentials over long-lived static
ones) limits how much damage a leaked credential can do before it's rotated. Resolve Ch 61's
deferred forward reference directly here: OIDC-based trusted publishing (already named in Ch 61 as
npm and PyPI's direction of travel) is a concrete instance of this — a short-lived, pipeline-scoped
identity is exchanged for a publish session with no persisted secret sitting anywhere at all,
which is strictly better than rotation of a long-lived token because there's no static credential
to rotate in the first place.

Cover secret storage as a decision, not a default: a dedicated secret manager or vault (issuing
short-lived, audited, dynamically generated credentials) against environment variables or config
files (static, longer-lived, and only as protected as whatever else can read the process's
environment or the filesystem). Take a position: a dedicated secret manager is the correct default
for anything beyond a single-developer local project, because it makes rotation and audit
mechanically possible rather than a manual, easily-skipped discipline — Principle 8 (mechanical
enforcement over human discipline) applied to credential handling specifically.

Anchor in real systems: a secret manager such as HashiCorp Vault or a cloud provider's native key
management service, issuing short-lived, dynamically generated database or service credentials
instead of static ones; OIDC trusted publishing as already named in Ch 61.

Do NOT cover: authentication protocol mechanics that consume a credential once issued (Ch 82 — this
chapter covers the credential material itself, not the protocol built on top of it); dependency or
package-signing trust (Ch 84); general logging strategy already covered in Ch 69 — this chapter
resolves only the specific mechanics of keeping a secret out of a log line, not what else belongs
in one.
```

---

## Ch 84 — Dependency Supply Chain Risk

```
This chapter closes Part XI by widening the aperture past code the team wrote at all, to code and
tooling the system merely depends on. It resolves two explicit forward references made by number:
Ch 63's deferred security threat model of a compromised or malicious dependency (explicitly
distinguished there from whether a dependency's version is current, which remains entirely Ch 63's
subject), and Ch 56's deferred full supply-chain threat model that tag and release signing defends
against.

Cover the threat model precisely: a dependency is code the team did not write and typically did
not fully review, pulled in via a package registry whose trust model is usually "anyone can publish
a package," and updated automatically or semi-automatically as part of routine maintenance
(Ch 63's subject). This is a different risk from an outdated dependency — the code can be exactly
current and still be malicious, compromised, or silently taken over from its original maintainer.

Cover the concrete attack patterns by name: a maintainer account or an entire package being
compromised and a malicious version published under the legitimate name (distinct from a
typosquatted package, which instead impersonates a legitimate package's name to trick a developer
into installing the wrong one); dependency confusion, where an internal, privately-named package is
shadowed by a public package of the same name uploaded to a public registry, tricking a build into
pulling the attacker's version instead of the internal one.

Cover the mechanical defenses and take a position consistent with Principle 8: a lockfile pinning
exact, hashed versions of every transitive dependency is the mechanical floor — it converts "trust
that nothing changed" into a verified fact the build system checks automatically, rather than a
discipline anyone has to remember to apply. Extend Ch 56's already-established tag-signing concept
one level further: package or artifact signing and provenance attestation (verifying not just that
a version number matches, but that the artifact was actually built by the expected pipeline from
the expected source) is the direct package-level analogue of the release-signing argument Ch 56
already made, referenced directly rather than re-derived. Name a Software Bill of Materials (SBOM)
as the mechanism for knowing what's actually in a build well enough to know what needs patching
when a vulnerability is disclosed in some far-downstream transitive dependency no one on the team
directly chose.

Anchor in a well-documented, widely taught real incident in the package-ecosystem supply chain —
a popular open-source package being compromised through a maintainer handoff or account takeover,
with a malicious dependency added that specifically targeted a narrow, high-value downstream
user — as the canonical case for why "the version is current" and "the version is trustworthy" are
different questions.

Do NOT cover: whether a dependency's version is current or how routine updates are scheduled
(Ch 63, already covers this — reference directly, don't re-derive dependency debt); the mechanics
of what an annotated, signed tag actually is (Ch 56, already defined — extend the concept to
packages, don't redefine signing itself); general secrets management for the credentials a build
pipeline uses to publish (Ch 83).
```
