# Ch 79 — Threat Modeling

**Prerequisites:** [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (partial failure, fail-fast), [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md) (reversibility and blast radius), [Authentication and Authorization Boundaries](../part03-api-design/ch24-authentication-authorization-boundaries.md) (confused deputy problem, zero-trust architecture), [Spec-First Development](../part06-engineering-process/ch46-spec-first-development.md) (Principle 10), [Distributed Tracing](../part09-observability/ch72-distributed-tracing.md)

**New vocabulary introduced:** asset, adversary, attack surface, STRIDE, threat model

**Key takeaways:**
- A **threat model** answers three questions before a line of implementation code is written: what's worth protecting (assets), who benefits from compromising it and what they're capable of (adversaries), and where trust assumptions change as data crosses a boundary (trust boundaries, attack surface). Every later chapter in this Part assumes this vocabulary rather than re-deriving it.
- [Strong Recommendation] Threat model at design time, not after deployment. This is Principle 10 — the cost of correcting a mistake grows with the distance between when it's made and when it's discovered — applied to security specifically, and the same argument Ch 46 already made for spec review: a trust-boundary mistake caught on a diagram costs a redraw; the same mistake caught in production costs a redesign under incident pressure.
- [Strong Recommendation] Treat every boundary crossing as untrusted by default. An "internal" network, an authenticated user, or a trusted vendor integration is not a reason to skip enumeration — it is the next trust boundary to examine.
- [Consensus] Use a structured taxonomy (STRIDE) rather than open-ended "think like an attacker" brainstorming. Structured review is repeatable and teachable; ad hoc review reliably finds the exotic vulnerability the reviewers already find interesting and misses the mundane one that real adversaries actually use.
- The 2013 Target breach is the canonical case: the compromised asset was never the intended target. A narrow, low-value trust boundary (a vendor billing portal) was the entry point to a catastrophic one, because the boundary between them was never explicitly examined.

---

Security controls are responses to threats, and a control is only as good as the understanding of what it defends and against whom. Many security failures do not trace back to weak cryptography or a coding mistake — they trace back to engineers protecting the wrong thing: a team hardens password storage while an administrative API sits unintentionally exposed to the internet, or ships careful authorization logic while diagnostic metadata leaks the internal topology that logic was supposed to hide. Threat modeling exists to catch this class of mistake before it's built, not to predict every possible attack.

Chapter 24 already introduced the **confused deputy problem** and **zero-trust architecture** as instances of trust-boundary reasoning applied to one specific boundary — service-to-service authorization. This chapter formalizes the general methodology those were specific applications of. It is not new territory; it is the vocabulary this whole Part is built on.

### Decision: Threat Model at Design Time or Rely on Post-Deployment Detection

**What it is:** Whether a system's security posture is analyzed structurally during specification and architecture — before implementation begins — or left to automated scanning (SAST/DAST), bug bounties, and penetration testing after the system is already running.

**Why it exists:** This is Principle 10 applied directly to security. An architectural flaw baked into a core protocol choice or a foundational trust model cannot be patched by a web application firewall or caught by a linter; correcting it after deployment means a low-reversibility rewrite of a system already carrying production traffic. Ch 46 made the same argument for spec review generally — catching a design flaw before code exists is categorically cheaper than catching it after. Threat modeling is that argument applied to the specific failure mode of an unexamined trust boundary.

**Options:**
- **Design-time modeling** — a structured trust-boundary walk (STRIDE or equivalent) during the design phase, before implementation starts.
- **Post-deployment detection** — rely on SAST/DAST tooling, bug bounties, and penetration tests to surface issues once the system exists.

**Trade-offs:** Design-time modeling lowers the blast radius of structural flaws and keeps security controls matched to the system's actual architecture, at the cost of upfront design friction and cross-team discipline that can slow early prototyping. Post-deployment detection maximizes day-one velocity and adds no early process overhead, but it guarantees that any structural flaw it does find arrives as accumulated risk discovered under incident conditions rather than as a design-review comment — and some categories of flaw (an unauthenticated internal API everyone assumed was unreachable) are exactly the kind that scanning tools are worst at surfacing, because nothing about them looks anomalous in isolation.

**When to choose each:** [Strong Recommendation] Design-time modeling for core infrastructure, data storage layers, multi-tenant boundaries, authentication pipelines, or anything where compromise is high-blast-radius and low-reversibility (Ch 09). Post-deployment detection alone is defensible only for isolated, non-critical prototypes and ephemeral experiments whose blast radius approaches zero — never as the primary strategy for anything that will carry real trust boundaries in production.

**Common failure modes:** *The unpatchable core.* A team builds a microservices architecture on an internal Kubernetes namespace, assuming the internal network is safe by virtue of being internal. Penetration testing after launch reveals that any compromised pod can sniff east-west traffic across the cluster, because no service authenticates its peers or encrypts internal transport. Fixing it means rewriting the transport layer of every production service, under active incident conditions, instead of choosing a service mesh with mTLS on day one.

**Example:** A design review that catches an unnecessarily internet-accessible administrative endpoint before implementation typically costs changing a diagram. Finding the same exposure after deployment costs architectural redesign, a migration plan, and possibly customer notification.

### Decision: Treat Every Boundary Crossing as Untrusted, or Trust the Perimeter

**What it is:** A **trust boundary** is any point where data or control passes between components operating under different trust assumptions — internet to application, service to service, an internal network to a build pipeline, a third-party vendor to internal infrastructure. The **attack surface** is everything reachable from outside a given trust boundary: every API, administrative interface, file-upload path, build pipeline, and diagnostic endpoint. The decision is whether every such crossing gets explicit sanitization and validation, or whether components inside a shared network or environment are allowed to trust each other implicitly.

**Why it exists:** As architectures decompose into more services, those services exchange growing volumes of diagnostic, performance, and transactional metadata. An engineer who assumes internal traffic is safe lets rich internal context — internal API responses, stack traces, service topology — cross boundaries unfiltered. Security problems concentrate at boundaries, not inside individual components, precisely because a boundary is where an assumption silently changes and nobody re-checks it.

**Options:**
- **Explicit boundary enumeration** — every crossing is treated as untrusted; inbound data is validated and outbound data (including metadata) is sanitized before it leaves the boundary.
- **Implicit perimeter trust** — components inside the same network or environment communicate transparently, trusting that whatever crosses an internal boundary is safe because the perimeter around all of them is defended.

**Trade-offs:** Explicit enumeration prevents internal topology, naming conventions, and infrastructure detail from leaking to anyone who can observe a boundary crossing, at the cost of writing and maintaining filtering and sanitization logic at every egress point. Implicit perimeter trust is simpler to build and debug — diagnostic and tracing data propagate transparently — but the entire model depends on the perimeter itself never being breached; the moment one internal component is compromised, everything downstream that trusted it implicitly is exposed with no second layer of defense.

**When to choose each:** [Strong Recommendation] Explicit boundary enumeration by default for any system with external clients, third-party integrations, or multiple tenants — anywhere the adversary model plausibly includes a malicious insider or a compromised adjacent tenant (Ch 80 makes the deeper architectural argument for defense in depth this generalizes to). Implicit trust is defensible only inside a genuinely single-tenant, hardware-isolated environment where the cost of sanitizing every crossing is measurably incompatible with a latency budget — a narrow exception, not a default.

**Common failure modes:** *The topology leak.* This resolves Ch 72's deferred forward reference: a distributed trace context propagated across a boundary is a concrete instance of attack surface, not merely a tracing-mechanics detail. A team instruments distributed tracing across twenty services and forwards `traceparent`/`tracestate` headers by default. Nobody strips them at the public edge, so an external client receives raw trace context in HTTP responses — internal service names, routing structure, and infrastructure conventions an attacker can use to move from a blind attack to a targeted one. The tracing mechanism was never the problem; crossing the boundary without considering what accompanied it was.

**Example:** Service meshes such as Istio enforce mutual TLS and header sanitization at every pod-to-pod boundary in a Kubernetes cluster, a deliberate rejection of the older assumption that traffic originating inside the data center firewall can be trusted by default.

### Decision: Walk Boundaries With a Structured Taxonomy, or Rely on Ad Hoc Review

**What it is:** Whether vulnerability discovery follows a systematic, exhaustive walk of every trust boundary using a named taxonomy, or relies on open-ended intuition and "think like an attacker" review.

**Why it exists:** Engineers naturally gravitate toward the code paths they understand best or the most technically interesting exploit — and correspondingly overlook the mundane, catastrophic ones. **STRIDE** (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege), popularized by Microsoft as a design-review framework, forces a reviewer to evaluate every category against every boundary instead of stopping once an interesting one is found.

**Options:**
- **STRIDE** — a matrix walk of all six categories against every enumerated trust boundary.
- **Ad hoc / intuitive review** — open-ended architectural walkthroughs driven by senior engineers' experience.

**Trade-offs:** STRIDE guarantees baseline coverage and eliminates blind spots on ordinary operational risks (a missing audit log, an unauthenticated internal endpoint), but can degenerate into a check-the-box exercise if applied mechanically without real engineering engagement. Ad hoc review is efficient and excellent at surfacing complex, domain-specific business-logic flaws, but provides no structural guarantee of completeness — it reliably misses whatever category the reviewers didn't think to look for.

**When to choose each:** [Consensus] STRIDE as the baseline for every architectural design review, giving every team a uniform, repeatable minimum bar. Ad hoc review adds real value only as a secondary pass on top of a completed STRIDE walk, hunting for the complex logic flaws a checklist can't be expected to catch.

**Common failure modes:** *The blind spot of intuition.* During an informal review of a new ledger service, senior engineers spend hours debating the cryptographic signature scheme on transactions — the interesting problem. Because there's no structured walk of the STRIDE matrix, nobody checks the **Repudiation** or **Denial of Service** categories, and the service ships writing audit logs to an unthrottled local disk with no rate limiting: any authenticated user can fill the disk and silently crash the host.

**Example:** Microsoft and AWS both mandate structured threat-modeling review inside their internal engineering workflows — no major feature reaches implementation until its design has been decomposed into data-flow diagrams and walked against a formal taxonomy.

### Identify Assets and Adversaries Before Choosing What to Protect

Enumerating boundaries and running STRIDE against them presumes two prior questions have already been answered: what's worth protecting, and from whom.

An **asset** is anything whose compromise would matter — not only databases and customer records, but credentials, signing keys, build infrastructure, source code, availability, and reputation. Security controls are expensive, and protecting every component equally is not possible; the value of a control tracks the value — and the blast radius (Ch 09) — of the asset it protects. A code-signing key is often a higher-value asset than the application source it signs: source code reveals implementation detail, but a compromised signing key lets an adversary distribute malicious software that appears to come from the legitimate publisher.

An **adversary** is any party that benefits from compromising an asset, characterized by realistic capability rather than a vague label like "hackers." An anonymous internet scanner, a malicious insider, and a resourced criminal group present entirely different risks, and modeling only the first while ignoring the second is a common and expensive omission — many ransomware incidents begin with stolen employee credentials, where the adversary is, for a while, indistinguishable from a legitimate user.

### Why Smart Engineers Disagree on the Adversary Model

The disagreement in a threat-modeling session is almost never about whether a given vulnerability exists. It's about whether the adversary that could exploit it is realistic enough to justify the cost of mitigating it.

Engineers optimizing for delivery pragmatism reason in expected value: probability times impact. If a mitigation requires real architectural indirection — application-level encryption on every internal database column, say — and the attack path requires a compromised cloud-root administrator credential, they'll argue to accept the risk rather than pay the complexity tax, on the grounds that engineering against an omnipotent adversary produces unmaintainable systems for no measurable benefit.

Engineers optimizing for durability reject probability-based reasoning here specifically. Under an assume-breach posture, a structural path that allows compromise will eventually be exploited, because an adversary is an adaptive, intelligent actor rather than a random hardware failure with a fixed failure rate — probability doesn't apply the same way it does to a failing disk. Leaving a boundary unmitigated because exploitation looks unlikely today reads, to this view, as a structural failure of engineering discipline, not an economic trade-off.

Neither position is wrong in isolation; the resolution is aligning the adversary model with the system's actual blast radius rather than importing a stance from a different system. A decentralized financial protocol or a medical-device telemetry pipeline has an unrecoverable blast radius and warrants modeling an advanced, persistent adversary. An internal ERP tool behind corporate SSO does not — paying the complexity tax to defend it against a nation-state interception capability is a mismatch, not rigor.

### Case Study: The 2013 Target Breach

```
[Attacker]
   │  phishing / stolen credentials
   ▼
[Fazio Mechanical — HVAC vendor billing portal]
   │  crosses an unsegmented internal boundary
   ▼
[Target's internal corporate network]
   │  lateral movement via Active Directory
   ▼
[Point-of-sale systems]
```

Attackers did not target Target's payment infrastructure directly. They compromised credentials for Fazio Mechanical, a third-party HVAC vendor with legitimate access to a billing and project-management portal — a narrow, low-value integration by any reasonable asset ranking. The structural failure was that Target's network treated that portal's segment as implicitly trusted by the rest of the corporate infrastructure: once the vendor credentials were compromised, the attacker crossed into the internal network with no additional boundary to stop lateral movement, reached Active Directory domain controllers, and from there deployed malware onto point-of-sale terminals, exfiltrating over 40 million card numbers.

The vendor credentials were never the target. They were the path across a trust boundary nobody had explicitly examined. A STRIDE walk of the boundary between the vendor portal and the internal network would have surfaced it directly as an Elevation of Privilege and Information Disclosure risk — the fix (isolating the vendor application with no routing path to the core network) touches one firewall rule. It is the kind of fix that is cheap on a design diagram and was, in production, catastrophic to have skipped.
