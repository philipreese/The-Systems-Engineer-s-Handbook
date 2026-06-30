# Chapter 10 — Monolith vs. Service Decomposition

**Prerequisites:** [Part I, Ch 03 — Coupling and Cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md), [Ch 06 — Cost Models and Mechanical Sympathy](../part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md), [Ch 07 — Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md), [Ch 08 — Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md). Specifically: the latency hierarchy, partial failure, the distributed monolith anti-pattern, and Conway's Law.

**New vocabulary introduced:** modular monolith, big ball of mud, strangler fig pattern

**Key takeaways:**
- Decomposition is not inherently good. It is a trade of local simplicity for the ability to scale, deploy, and fail independently — and that trade costs real latency, real operational overhead, and real coordination effort. Most systems should start, and many should stay, as a monolith.
- Extraction is justified by a specific, named constraint — a component with a genuinely different scaling profile, a genuinely different failure domain, or an organization that has outgrown single-codebase coordination — not by the belief that microservices are the mature end state.
- The distributed monolith — services that are deployed independently but still coupled through synchronous chains and shared schemas — is the dominant failure mode of decomposition. It carries the operational cost of distribution with none of the autonomy benefit.
- When extraction is justified, do it incrementally, with the strangler fig pattern, against live traffic. Big-bang rewrites of production-critical systems fail more often than they succeed.

---

## Start as a Monolith

**What it is:** A monolith is a system where the core components execute within a single deployable unit, sharing a runtime and (usually) a data boundary. A **modular monolith** enforces strict internal module boundaries and information hiding (Ch 04) within that single unit; a **big ball of mud** has none, and every component can reach into every other.

**Why it exists:** A monolith minimizes coordination overhead, eliminates network failure modes for internal calls, and lets the compiler and the database enforce consistency natively. It is the lowest-complexity architecture available, not a legacy stage to be outgrown.

**Options:**
1. **Big ball of mud** — no internal boundaries; everything calls everything
2. **Modular monolith** — single deployable unit, strictly enforced internal module boundaries
3. **Service decomposition** — independently deployable units communicating over a network

**Trade-offs:**

| Architecture | Buys | Costs |
|---|---|---|
| Big ball of mud | Maximum initial velocity | Degenerates into unpredictable regressions as any change can touch anything |
| Modular monolith | Fast development, simple deployment, linear stack traces, native ACID transactions across entities | Deployment coupling — every change ships the whole system together |
| Service decomposition | Independent scaling, deployment, and failure domains | Network latency tax (Ch 06) on every cross-boundary call, partial failure (Ch 07), real operational overhead |

**When to choose each:**
- *Modular monolith:* the default starting architecture for nearly every new system, and the correct steady state for systems where transactional integrity across entities matters and load is roughly uniform.
- *Big ball of mud:* never deliberately. It is what a modular monolith becomes when module boundaries are not enforced.
- *Service decomposition:* only once a specific constraint (below) makes the monolith more expensive than the coordination overhead of distribution.

**Common failure modes:**
- **Deployment gridlock:** a single bad commit blocks the release pipeline for the entire organization — the predictable cost of deployment coupling in a monolith that has grown past the team size it was sized for.
- Treating decomposition as a maturity ladder, splitting services as a default next step rather than in response to an observed constraint.

**Example:** Stack Overflow ran billions of monthly requests on a deliberately monolithic architecture, leaning on mechanical sympathy (Ch 06) — in-memory caching and single-process execution — to serve enormous load from a small server footprint, specifically to avoid the network latency tax. Shopify makes the same bet at much larger scale: its core commerce engine is a modular monolith, kept unified to preserve transactional integrity and operational simplicity, with services extracted only where a specific constraint demands it. **[Strong Recommendation: default to a modular monolith; treat service decomposition as something you earn, not something you start with]**

---

## Decomposition by Constraint

**What it is:** The deliberate extraction of a piece of functionality out of the monolith into an independently deployed, networked service, in exchange for accepting the latency and operational tax of distribution.

**Why it exists:** A monolith forces every component to scale, deploy, and fail together. That is efficient until one component's requirements diverge sharply enough from the rest that uniform treatment becomes the more expensive option.

**Options:**
1. **Decomposition by scale profile** — extracting a component whose resource demands (compute, memory, I/O) differ fundamentally from the rest of the system
2. **Decomposition by failure domain** — extracting a critical path so it can survive the collapse of a less reliable subsystem
3. **Decomposition by organizational boundary** — extracting a service to give a team deployment autonomy, per Conway's Law (Ch 08)

**Trade-offs:**
- *Scale profile / failure domain:* fixes a real, measurable constraint and can mathematically improve system-level reliability or cost — but forces every caller to now reason about partial failure and distributed state.
- *Organizational boundary:* maximizes a team's local deploy velocity — a team can ship ten times a day without cross-team coordination — but the global system gains a network boundary it didn't need for technical reasons, and that boundary still has to be paid for in latency and failure handling.

**When to choose each:**
- A subsystem has a measurably different scaling profile from the rest of the system — for example, a GPU-bound image-processing pipeline sitting behind an otherwise ordinary CRUD application.
- A subsystem has a different reliability requirement than its neighbors — for example, checkout needs to survive the recommendation engine being down.
- The engineering organization has grown large enough that a single shared codebase and deploy pipeline is the actual bottleneck on shipping, not the architecture.

**Common failure modes:**
- **The distributed monolith** (Ch 03) is the dominant failure mode here: services are split, but they still share a database schema or call each other synchronously in long chains, so a schema change or an outage still requires coordinating every service at once. It has the operational cost of distribution with none of the autonomy it was meant to buy.
- **Premature extraction:** splitting a service based on a theoretical future load imbalance rather than an observed one, paying the distribution tax for a constraint that hasn't materialized.
- **Resume-driven decomposition:** adopting services because they are the current industry default, not because a constraint demands them.

**Example:** Several of Netflix's and eBay's early microservice migrations initially produced exactly this distributed-monolith failure: services existed as separate deployables, but were chained together through synchronous calls tightly enough that a slowdown anywhere amplified into a slowdown everywhere. Both stabilized only after introducing asynchronous boundaries (Ch 17) and enforcing that each service own its own data (Ch 18). Amazon and Netflix are the cases usually cited *for* decomposition, and the constraint that actually justified it was organizational: thousands of engineers could no longer ship through one repository and one deploy pipeline. **[Consensus: extraction without an independent failure domain and independent data ownership reproduces monolithic coupling at network latency]**

---

## The Strangler Fig Pattern: Safe Extraction

**What it is:** An incremental migration pattern, named by Martin Fowler, in which a new service is grown around the edge of an existing monolith — intercepting and handling specific traffic — until the corresponding functionality can be removed from the monolith entirely.

**Why it exists:** A "big bang" rewrite — pausing feature work to rebuild the system as services before shipping anything — is a high-risk bet that the new architecture will reach functional parity before the business runs out of patience or budget. Most big-bang rewrites of production-critical systems do not finish.

**Options:**
1. **Big bang rewrite** — halt the monolith, build the replacement architecture from scratch, cut over
2. **Strangler fig pattern** — route a thin, increasing slice of live traffic to the new service via an edge gateway while the monolith keeps handling the rest

**Trade-offs:**
- *Big bang rewrite:* in principle produces a clean architecture unconstrained by legacy decisions; in practice is a high-variance gamble that rarely survives contact with a live system.
- *Strangler fig:* low risk, validated incrementally against production traffic — but it means running two architectures simultaneously for an extended period, duplicating some data, and maintaining temporary routing logic that has to eventually be deleted.

**When to choose each:**
- *Strangler fig:* the default for decomposing any system that is already serving production traffic.
- *Big bang rewrite:* only when the legacy system is genuinely unmaintainable, legally compromised, or being fully retired rather than replaced piece by piece.

**Common failure modes:**
- **The permanent strangler:** the team extracts the easy, stateless pieces — notification sending, document generation — and stalls before reaching the tangled core around the primary database. The organization ends up paying the operational tax of new services while still maintaining the full weight of the legacy monolith, the worst of both architectures rather than either one.

**Example:** Extracting a billing service: stand up the new service, configure the edge gateway to route 1% of `/checkout` traffic to it while 99% still goes to the monolith, watch error rates, ramp gradually to 100%, then delete the legacy billing code path. The monolith and the new service are both live and both correct throughout the migration — there is no cutover moment where the system is down or wrong.

---

## Why Smart Engineers Disagree

The disagreement here is not about whether monoliths or microservices are "better" — it's about where the extraction threshold sits, and it tracks how each side weighs the cost of getting boundaries wrong against the cost of paying the distribution tax early.

Engineers who point to Amazon or Netflix argue that decomposing late is the expensive mistake: extracting services after a monolith has calcified is exponentially harder than building decoupled from day one, so it's worth paying the distributed-systems tax upfront to avoid a painful migration later. This view is most common among engineers who have lived through an organization actually hitting the coordination ceiling a monolith imposes on a large team.

Engineers who argue for monolith-first point out that a system decomposed before its domain boundaries are understood will have its boundaries drawn wrong — and a wrongly decomposed system is a distributed monolith, which combines the cost of distribution with none of the benefit. This view is most common among engineers who have lived through the alternative failure: a service split made on guesses, requiring a "macro-refactor" back into a single deployable once the guess turned out wrong.

Both are reasoning correctly about real risk; they disagree about which risk is more expensive to realize *first*. The synthesis most production systems converge on: the monolith is not a stage to escape, it's the mechanism by which the true domain boundaries get discovered under real traffic. Decomposition becomes an engineering exercise instead of a speculative guess only after those boundaries have been proven — which is why Shopify and Stack Overflow defer it indefinitely, and why Amazon and Netflix only made the cut once organizational scale gave them an unambiguous, named constraint.

*Concepts expanded in later chapters: internal structural patterns within a service (Part II, Ch 11), the dependency inversion mechanics behind swappable infrastructure (Part II, Ch 12), API versioning across service boundaries (Part II, Ch 16), data ownership patterns (Part II, Ch 18), synchronous vs. asynchronous communication mechanics (Part II, Ch 17).*
