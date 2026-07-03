# Appendix B — Common Engineering Smells

This appendix is an index, not a chapter. It catalogs the mundane, observable signals an engineer actually notices — in a pull request, a dashboard, a standup — that a decision from one of the 90 chapters may have been made poorly, or not made deliberately at all.

A smell is a prompt to investigate, not a verdict. Every entry below includes a **false positive**: a specific, concrete situation where the exact same observable symptom is the correct result of a deliberate trade-off this handbook itself argues for. Treating a smell as an automatic defect regardless of context is its own failure mode — the mechanical version of ignoring Chesterton's fence. Use this appendix to know where to look, not to skip the judgment call the linked chapter actually walks through.

---

## 1. Architecture & Coupling Smells

**The Shared-Database Schema Mutation Outage**
*What you observe:* Renaming a column or altering a table in the primary database causes an immediate runtime failure in a completely separate service that wasn't touched by the pull request.
*Usually a symptom of:* Two services sharing a database instead of each owning its own — a distributed monolith at the data layer.
*Full treatment:* [Ch 10 — Monolith vs. Service Decomposition](../part02-software-architecture/ch10-monolith-vs-service-decomposition.md), [Ch 18 — Data Ownership Boundaries](../part02-software-architecture/ch18-data-ownership-boundaries.md)
*False positive:* A deliberate modular monolith, deployed as a single compilation unit, where the compiler enforces internal boundaries at build time and a shared schema carries no runtime coupling risk.

**The Persistent Strangler Stub**
*What you observe:* An architecture diagram shows 95% of traffic routed through new services, but a permanent, unowned 5% slice still hits a legacy core nobody will touch.
*Usually a symptom of:* A stalled strangler-fig migration, now paying the full operational cost of two competing architectures indefinitely.
*Full treatment:* [Ch 10 — Monolith vs. Service Decomposition](../part02-software-architecture/ch10-monolith-vs-service-decomposition.md) (the strangler fig pattern)
*False positive:* A deliberate, permanent anti-corruption layer isolating a clean internal domain from an immutable third-party system that was never meant to be fully replaced.

**The Cross-Layer Pull Request**
*What you observe:* A single user-visible feature requires touching UI, business logic, persistence, and infrastructure code across dozens of files.
*Usually a symptom of:* Poor boundary placement — a change local to one concern has become a change to the whole system.
*Full treatment:* [Ch 03 — Coupling and Cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md), [Ch 13 — Coupling and Cohesion at the Architecture Level](../part02-software-architecture/ch13-coupling-cohesion-architecture-level.md)
*False positive:* A deliberate, cross-cutting migration — introducing structured logging or replacing a serialization library — that is expected to touch everything exactly once.

**Circular Dependency Appears**
*What you observe:* The build system reports an import cycle between two modules that previously built independently.
*Usually a symptom of:* Responsibility has drifted until neither module can be said to own the thing the other now depends on.
*Full treatment:* [Ch 12 — Dependency Direction and Inversion](../part02-software-architecture/ch12-dependency-direction-inversion.md)
*False positive:* A short-lived cycle introduced mid-refactor and resolved before the branch merges.

---

## 2. API Design Smells

**The Multi-Flag God Endpoint**
*What you observe:* A single request object accumulates more than a handful of optional boolean flags (`include_history`, `skip_validation`, `dry_run`) that interact in ways no caller fully understands.
*Usually a symptom of:* A flat API surface standing in for what should be layered, progressively disclosed capabilities.
*Full treatment:* [Ch 15 — API Surface Design](../part02-software-architecture/ch15-api-surface-design-expose-hide.md)
*False positive:* An internal, throughput-first data pipeline where ergonomics are explicitly secondary to raw transfer efficiency.

**The Tightened Validation Input Trap**
*What you observe:* Adding a stricter validation rule to an existing field — a tighter bound on an age field, say — causes existing clients to start receiving errors they never got before.
*Usually a symptom of:* Treating a contract tightening as a minor fix instead of the breaking change it actually is.
*Full treatment:* [Ch 16 — Versioning and Backward Compatibility](../part02-software-architecture/ch16-versioning-backward-compatibility.md)
*False positive:* An internal RPC surface where every caller lives in the same monorepo and deploys in lockstep with the change.

**The Database Schema Reflection in JSON**
*What you observe:* A public API response includes an internal surrogate key, an ORM-generated field name, or a `deleted_at` column straight from the table definition.
*Usually a symptom of:* No boundary mapping — internal representation is passed straight through instead of hidden behind an explicit contract.
*Full treatment:* [Ch 04 — Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Ch 15 — API Surface Design](../part02-software-architecture/ch15-api-surface-design-expose-hide.md)
*False positive:* A throwaway internal tool where the same person owns both the producer and the one consumer, deployed together.

**One Endpoint Returns Different Shapes**
*What you observe:* The same endpoint returns structurally different payloads depending on a hidden flag or server-side condition the caller can't see in advance.
*Usually a symptom of:* The interface no longer represents one contract — several different operations have been collapsed into one route.
*Full treatment:* [Ch 20 — Resource Modeling](../part03-api-design/ch20-resource-modeling.md)
*False positive:* A GraphQL endpoint, where the caller explicitly requests the shape it gets back.

---

## 3. Code Organization Smells

**The Pass-Through Service Layer**
*What you observe:* A method's entire body is an unmodified call to the layer beneath it — `getUser(id)` that does nothing but `return userRepository.getUser(id)`.
*Usually a symptom of:* A layer added without a volatile decision to hide behind it — pure indirection tax with nothing purchased in return.
*Full treatment:* [Ch 14 — Abstraction Layers: When to Add One](../part02-software-architecture/ch14-abstraction-layers-when-to-add-one.md), [Ch 31 — When Abstractions Help vs. When They Obscure](../part04-code-organization/ch31-when-abstractions-help-vs-when-they-obscure.md)
*False positive:* A strictly enforced layered architecture where every tier must go through the adjacent one, and the pass-through exists to keep that invariant mechanically true even when today's implementation is trivial.

**The Core Business Logic SDK Import**
*What you observe:* A file computing domain rules — tax calculation, pricing — directly imports a cloud provider's SDK or a specific database driver.
*Usually a symptom of:* A callee-owned interface leaking vendor vocabulary straight into logic that should only speak the business's own language.
*Full treatment:* [Ch 12 — Dependency Direction and Inversion](../part02-software-architecture/ch12-dependency-direction-inversion.md)
*False positive:* Importing a genuinely foundational, stable primitive — a language's standard I/O interface — that is never going to be swapped out.

**Utility Folder Growth**
*What you observe:* `utils/`, `helpers/`, or `common/` becomes one of the largest, least-owned directories in the repository.
*Usually a symptom of:* Unrelated responsibilities accumulating in a place with no cohesion, because nobody had to decide where they actually belonged.
*Full treatment:* [Ch 27 — File and Module Structure](../part04-code-organization/ch27-file-and-module-structure.md), [Ch 28 — Naming Conventions and When They Matter](../part04-code-organization/ch28-naming-conventions-and-when-they-matter.md)
*False positive:* A language runtime or standard library deliberately collecting broadly reusable, genuinely general-purpose primitives.

**One Class Has Hundreds of Imports**
*What you observe:* A single class directly depends on dozens of unrelated modules — validators, formatters, persistence, orchestration — all at once.
*Usually a symptom of:* A class that has quietly become several classes wearing one name.
*Full treatment:* [Ch 03 — Coupling and Cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md), [Ch 29 — When to Split Files vs. Keep Together](../part04-code-organization/ch29-when-to-split-files-vs-keep-together.md)
*False positive:* A composition root or dependency-injection bootstrap file, whose entire job is wiring together everything else exactly once.

---

## 4. Testing Smells

**The Docker-Dependent Domain Unit Test**
*What you observe:* Verifying a pure business rule — a discount calculation — requires spinning up Postgres and Redis containers and takes seconds instead of milliseconds.
*Usually a symptom of:* Business logic that was never actually decoupled from the infrastructure it happens to run alongside.
*Full treatment:* [Ch 11 — Layered, Hexagonal, and Ports-and-Adapters Architecture](../part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md), [Ch 36 — When to Mock vs. Use Real Dependencies](../part05-testing-strategy/ch36-when-to-mock-vs-use-real-dependencies.md)
*False positive:* A dedicated integration or smoke-test suite whose entire purpose is validating real database behavior under real conditions.

**The Mock Museum**
*What you observe:* A test needs fifty lines of stub configuration to verify a three-line method.
*Usually a symptom of:* An abstraction layer with no real complexity behind it, being paid for in test setup instead of production code.
*Full treatment:* [Ch 14 — Abstraction Layers: When to Add One](../part02-software-architecture/ch14-abstraction-layers-when-to-add-one.md), [Ch 36 — When to Mock vs. Use Real Dependencies](../part05-testing-strategy/ch36-when-to-mock-vs-use-real-dependencies.md)
*False positive:* Testing an anti-corruption layer built specifically to translate a large, genuinely messy third-party contract — the stubbing volume reflects the vendor's real complexity, not an unnecessary layer.

**Tests Retry Until They Pass**
*What you observe:* CI is configured to automatically rerun failing tests, and a merge proceeds once a retry eventually succeeds.
*Usually a symptom of:* Flakiness has been normalized into pipeline policy instead of investigated as a real bug.
*Full treatment:* [Ch 57 — What Belongs in CI (and What Doesn't)](../part07-git-and-delivery/ch57-what-belongs-in-ci-and-what-doesnt.md)
*False positive:* A transient, external infrastructure outage — a cloud provider blip — genuinely outside the team's control, with the retry scoped narrowly to that dependency.

**Coverage Increases While Bugs Continue**
*What you observe:* The coverage percentage climbs sprint over sprint, but the rate of escaped defects doesn't move.
*Usually a symptom of:* Coverage has become the target instead of the diagnostic it was meant to be — Principle 9 in practice.
*Full treatment:* [Ch 41 — Coverage: What It Measures and What It Doesn't](../part05-testing-strategy/ch41-coverage-what-it-measures-and-what-it-doesnt.md)
*False positive:* The early phase of building a regression suite for a codebase that had no automated tests at all — a rising floor with no history of prior coverage to compare bug rates against.

---

## 5. Engineering Process Smells

**The Speculative Technology Migration Preparation**
*What you observe:* A design doc adds an abstraction layer so the storage engine can theoretically switch from Postgres to MongoDB "just in case," with no profiling data suggesting the current database is a constraint.
*Usually a symptom of:* Paying an indirection tax today for optionality that has not been shown to be needed.
*Full treatment:* [Ch 14 — Abstraction Layers: When to Add One](../part02-software-architecture/ch14-abstraction-layers-when-to-add-one.md) ("just in case" layers), [Ch 85 — When to Optimize (and When Not To)](../part12-performance/ch85-when-to-optimize.md)
*False positive:* A real anti-corruption boundary built around a vendor integration whose contract is already, concretely scheduled for replacement.

**The Resume-Driven Microservice Sweep**
*What you observe:* A five-person team splits a low-traffic internal tool into a dozen services, taking on distributed tracing and deployment coordination overhead in the process.
*Usually a symptom of:* Decomposition driven by industry trend rather than an actual organizational or scaling constraint.
*Full treatment:* [Ch 08 — Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md) (Conway's Law), [Ch 10 — Monolith vs. Service Decomposition](../part02-software-architecture/ch10-monolith-vs-service-decomposition.md)
*False positive:* Isolating one genuinely distinct workload — a GPU-bound image-processing job — away from an otherwise ordinary stateless application, for a real hardware reason.

**The Post-Hoc Benchmark**
*What you observe:* An engineer ships a complex, hard-to-read performance rewrite, then writes a small benchmark against a toy dataset afterward to justify the pull request.
*Usually a symptom of:* Optimization performed before evidence, with the evidence manufactured retroactively to fit a decision already made.
*Full treatment:* [Ch 85 — When to Optimize (and When Not To)](../part12-performance/ch85-when-to-optimize.md), [Ch 86 — Profiling-First](../part12-performance/ch86-profiling-first.md)
*False positive:* Building a foundational, shared data structure or lock-free primitive inside a platform runtime, where mechanical sympathy is a legitimate day-one design constraint rather than a speculative one.

**Design Reviews Happen After Implementation**
*What you observe:* The first real architectural discussion of a feature happens in a PR review, after thousands of lines are already written.
*Usually a symptom of:* Design validation moved to the most expensive point in the process to discover a flaw.
*Full treatment:* [Ch 46 — Spec-First Development](../part06-engineering-process/ch46-spec-first-development.md)
*False positive:* An exploratory prototype whose explicit purpose is discovering unknown constraints before a design is even attempted.

---

## 6. Git and Delivery Smells

**The Multi-Repo Lockstep Pull Request**
*What you observe:* One feature ticket requires three pull requests across three repositories, merged and deployed in a specific, coordinated order.
*Usually a symptom of:* Services that were split for deployment independence but never actually achieved it — a distributed monolith at the delivery layer.
*Full treatment:* [Ch 10 — Monolith vs. Service Decomposition](../part02-software-architecture/ch10-monolith-vs-service-decomposition.md), [Ch 18 — Data Ownership Boundaries](../part02-software-architecture/ch18-data-ownership-boundaries.md)
*False positive:* A coordinated emergency security patch that genuinely must land across several shared libraries at once.

**The Blocked Monolith Release Train**
*What you observe:* One engineer's unrelated bug halts the entire production deployment pipeline for every team.
*Usually a symptom of:* An organization that has outgrown the coordination model of a single, un-segmented release process.
*Full treatment:* [Ch 08 — Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md) (Conway's Law), [Ch 10 — Monolith vs. Service Decomposition](../part02-software-architecture/ch10-monolith-vs-service-decomposition.md)
*False positive:* An early-stage team that has deliberately accepted deployment coupling in exchange for maximum day-one iteration speed.

**The Permanent Feature Branch**
*What you observe:* A branch stays open for months, diverging further from main every week, while everyone waits for a "big bang" rewrite to be validated.
*Usually a symptom of:* Integration risk accumulating instead of being paid down incrementally.
*Full treatment:* [Ch 50 — Branching Strategies](../part07-git-and-delivery/ch50-branching-strategies.md)
*False positive:* A research prototype or experimental rewrite deliberately kept separate from the production delivery pipeline, with no intention of a future merge.

**Emergency Commits Directly to Main**
*What you observe:* Production fixes routinely skip normal review and land straight on the default branch.
*Usually a symptom of:* The ordinary review process isn't fast enough for the situations that actually matter, so it gets bypassed instead of adapted.
*Full treatment:* [Ch 47 — Code Review](../part06-engineering-process/ch47-code-review.md)
*False positive:* A controlled incident response, with the bypass logged and a real follow-up review scheduled — not a silent, permanent habit.

---

## 7. Documentation Smells

**The Documentation-Only Deprecation**
*What you observe:* A wiki page says an API version is deprecated, but traffic metrics show it's still serving nearly half of production requests.
*Usually a symptom of:* Treating prose as an operational control instead of building an actual, enforced sunset lifecycle.
*Full treatment:* [Ch 16 — Versioning and Backward Compatibility](../part02-software-architecture/ch16-versioning-backward-compatibility.md) (the sunset pattern)
*False positive:* The narrow window between documenting a newly discovered issue and an automated patch pipeline finishing its rollout.

**The Outdated Architecture Legend**
*What you observe:* The system diagram shows clean service isolation, but the actual code makes direct cross-database joins the diagram doesn't mention.
*Usually a symptom of:* Documentation that stopped being updated the moment the system it described started changing.
*Full treatment:* [Ch 66 — Keeping Documentation Honest](../part08-documentation/ch66-keeping-documentation-honest.md), [Ch 18 — Data Ownership Boundaries](../part02-software-architecture/ch18-data-ownership-boundaries.md)
*False positive:* A deliberately aspirational diagram from an early design workshop, clearly labeled as the target state rather than the current one.

**The Runbook That Starts With "Investigate"**
*What you observe:* The first step in an on-call guide is effectively "figure out what's wrong" rather than a concrete action.
*Usually a symptom of:* The document captures background knowledge instead of an operational procedure.
*Full treatment:* [Ch 68 — Runbooks and Operational Documentation](../part08-documentation/ch68-runbooks-and-operational-documentation.md)
*False positive:* A genuinely new, previously unseen class of incident, where investigation really is the correct first step.

**Wiki Search Returns Conflicting Answers**
*What you observe:* Two different pages describe two different deployment processes for the same system.
*Usually a symptom of:* No single, authoritative source of truth — every page is somebody's snapshot of the truth at the time they wrote it.
*Full treatment:* [Ch 66 — Keeping Documentation Honest](../part08-documentation/ch66-keeping-documentation-honest.md)
*False positive:* Documentation that intentionally maintains separate pages for multiple, currently-supported product versions.

---

## 8. Observability Smells

**The Trace-Isolation Guesswork Blindspot**
*What you observe:* An engineer stares at a distributed trace dashboard trying to guess which specific line of code, inside one service, is slow.
*Usually a symptom of:* Using the wrong tool for the question — a trace shows which service is slow, not why that service is slow internally.
*Full treatment:* [Ch 72 — Distributed Tracing](../part09-observability/ch72-distributed-tracing.md), [Ch 86 — Profiling-First](../part12-performance/ch86-profiling-first.md)
*False positive:* Using the same trace correctly to evaluate cross-service network transit time or queue delay — the question a trace is actually built to answer.

**The Blind Average Metric Illusion**
*What you observe:* A dashboard shows a healthy 50ms average latency while support tickets pile up about multi-second stalls.
*Usually a symptom of:* Measuring the mean when the thing that matters — and the thing users experience — is the tail.
*Full treatment:* [Ch 87 — Latency vs. Throughput Trade-offs](../part12-performance/ch87-latency-vs-throughput.md) (tail latency), [Ch 73 — Error Budgets and SLOs](../part09-observability/ch73-error-budgets-and-slos.md)
*False positive:* Monitoring aggregate throughput for a fully asynchronous, decoupled background pipeline where no individual request has a user waiting on it.

**The Local Profiler Micro-Stall Illusion**
*What you observe:* A flame graph from an instrumenting profiler says a small string-parsing function is 85% of execution time; replacing it changes nothing in production.
*Usually a symptom of:* The observer effect — the profiler's own overhead distorted the very timing it was measuring.
*Full treatment:* [Ch 86 — Profiling-First](../part12-performance/ch86-profiling-first.md)
*False positive:* Using an instrumenting profiler deliberately, in an offline, single-threaded investigation where exact call counts are genuinely the question being asked.

**Every Log Line Says "Starting"**
*What you observe:* Logs are full of lifecycle messages and nearly empty of business identifiers, outcomes, or request context.
*Usually a symptom of:* Logging written for the developer who wrote the code, not the operator debugging it at 2 a.m.
*Full treatment:* [Ch 69 — Logging: What to Log and at What Level](../part09-observability/ch69-logging-what-to-log-and-at-what-level.md)
*False positive:* Very early bootstrap code, running before any request context exists to attach to a log line.

---

## 9. Concurrency Smells

**The Unbounded Buffer Queue Expansion**
*What you observe:* Memory climbs steadily during a downstream stall, then the process dies in a sudden out-of-memory crash.
*Usually a symptom of:* A queue with no capacity limit, absorbing load instead of applying backpressure.
*Full treatment:* [Ch 01 — What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md) (buffer bloat), [Ch 08 — Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md) (Little's Law, backpressure)
*False positive:* A deliberately disk-backed, durable queue (Kafka-style) designed to absorb hours of backlog by design, not by accident.

**The Multi-Tenant Thread-Starvation Lock**
*What you observe:* Tail latency degrades sharply under load, and thread dumps show most workers blocked waiting on one global mutex.
*Usually a symptom of:* Lock granularity set far coarser than the actual contention pattern requires.
*Full treatment:* [Ch 75 — Locks: When to Use Them](../part10-concurrency/ch75-locks-when-to-use-them.md)
*False positive:* Deliberately serializing access to a genuinely singleton resource — a piece of hardware or an OS-level handle that cannot be accessed concurrently no matter how it's wrapped.

**Sleep Added to "Fix" a Race Condition**
*What you observe:* Production code contains a delay whose only purpose is giving another thread time to finish first.
*Usually a symptom of:* A real data race being papered over with timing instead of fixed with actual synchronization.
*Full treatment:* [Ch 74 — Shared State vs. Message Passing](../part10-concurrency/ch74-shared-state-vs-message-passing.md) (data race)
*False positive:* An intentional, documented backoff delay when retrying a call to a rate-limited external system.

**The Redundant Event Fan-Out Echo**
*What you observe:* One user action produces fifty overlapping, near-duplicate events on a message bus, and every consumer has to filter them back down.
*Usually a symptom of:* Weak bounded-context boundaries — several producers all believe they own the responsibility to announce the same change.
*Full treatment:* [Ch 13 — Coupling and Cohesion at the Architecture Level](../part02-software-architecture/ch13-coupling-cohesion-architecture-level.md) (event-driven decoupling), [Ch 17 — Synchronous vs. Asynchronous Communication](../part02-software-architecture/ch17-sync-vs-async-communication.md)
*False positive:* A deliberate broadcast, feeding several independent read-optimized projections (CQRS) that are each expected to receive the same event.

---

## 10. Security Smells

**Secrets in Source Control**
*What you observe:* An API key, password, or certificate turns up in the repository's history, even if it's since been removed from the current tip.
*Usually a symptom of:* Secret management bypassed for convenience during initial setup or a rushed fix.
*Full treatment:* [Ch 83 — Secrets Management](../part11-security/ch83-secrets-management.md)
*False positive:* A deliberately fake, documented test credential committed for a public example, that was never a real secret at all.

**The Hard-Coded Network Trust Model**
*What you observe:* A service drops input validation on its own endpoints because "it's behind the internal network, so traffic is safe."
*Usually a symptom of:* Treating a network boundary as an already-examined trust boundary instead of the next one to check.
*Full treatment:* [Ch 79 — Threat Modeling](../part11-security/ch79-threat-modeling.md), [Ch 80 — Defense in Depth](../part11-security/ch80-defense-in-depth.md)
*False positive:* A tightly coupled, single-process monolith where a "boundary" is actually just a function call verified by the compiler's own type system — there is no network boundary to have skipped examining.

**Permissions Keep Expanding**
*What you observe:* A service account's permission set only ever grows across changes, never shrinks.
*Usually a symptom of:* Least privilege maintained at grant time but never revisited afterward.
*Full treatment:* [Ch 82 — Authentication vs. Authorization](../part11-security/ch82-authentication-vs-authorization.md)
*False positive:* Early platform bootstrap, before responsibilities have been partitioned enough to know what the eventual minimal set should be.

**The Indefinitely Deferred Dependency Upgrade**
*What you observe:* A security advisory stays open release after release because the fix is "too risky to schedule."
*Usually a symptom of:* A dependency graph too brittle to upgrade safely, which is itself the real problem the deferral is hiding.
*Full treatment:* [Ch 84 — Dependency Supply Chain Risk](../part11-security/ch84-dependency-supply-chain-risk.md)
*False positive:* Deliberately waiting for a vendor's next release after their first attempted fix shipped its own regression.

---

## 11. Performance Smells

**The N+1 Network Loop**
*What you observe:* A database or request tracer shows one page load executing a hundred near-identical sequential queries or RPCs inside a loop.
*Usually a symptom of:* An access-pattern algorithm mistake — a sequence of round trips substituting for one batched or joined call.
*Full treatment:* [Ch 89 — Data Structure and Algorithm Selection in Practice](../part12-performance/ch89-data-structure-and-algorithm-selection.md), [Ch 04 — Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Ch 18 — Data Ownership Boundaries](../part02-software-architecture/ch18-data-ownership-boundaries.md)
*False positive:* A deliberate, single-row-at-a-time write path for a financial ledger, where individual transactional visibility is a correctness requirement, not an oversight.

**The Toy-Dataset Warm-Cache Illusion**
*What you observe:* A local micro-benchmark reports a function taking nanoseconds; the same code under production data and concurrency takes orders of magnitude longer.
*Usually a symptom of:* A benchmark small enough to live entirely in cache, hiding the memory-access cost that dominates at real scale.
*Full treatment:* [Ch 86 — Profiling-First](../part12-performance/ch86-profiling-first.md), [Ch 89 — Data Structure and Algorithm Selection in Practice](../part12-performance/ch89-data-structure-and-algorithm-selection.md)
*False positive:* A pure, self-contained algorithmic unit test whose entire purpose is checking correctness, not performance.

**The Thundering Herd Database Collapse**
*What you observe:* The instant a popular cache key expires, database CPU usage spikes to saturation within seconds.
*Usually a symptom of:* A cache stampede — many concurrent requests all regenerating the same expired entry at once, with no coalescing or lock in place.
*Full treatment:* [Ch 88 — Caching: Layers, Invalidation, Failure Modes](../part12-performance/ch88-caching.md)
*False positive:* A deliberate write-through path where every mutation is expected to reach the database immediately regardless of cache state.

**Performance Optimizations With No Measurement Attached**
*What you observe:* A pull request introduces a materially more complex implementation for the sake of speed, with no benchmark, profile, or production number attached to justify it.
*Usually a symptom of:* An assumed bottleneck standing in for a measured one.
*Full treatment:* [Ch 85 — When to Optimize (and When Not To)](../part12-performance/ch85-when-to-optimize.md), [Ch 86 — Profiling-First](../part12-performance/ch86-profiling-first.md)
*False positive:* An emergency mitigation shipped during an active, already-measured production incident, with formal benchmarking following once the fire is out.
