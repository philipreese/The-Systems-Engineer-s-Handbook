# Appendix B — Common Engineering Smells

## 1. Architecture & Coupling Smells

### Cross-Layer Pull Request

* **What you actually observe:** A feature PR modifies UI code, business logic, persistence, messaging, and infrastructure across dozens of files just to add one user-visible capability.
* **What it's usually a symptom of:** High coupling and poor boundary placement. A local change has become a system-wide change.
* **Where the real treatment lives:** The chapters on module boundaries, coupling and cohesion, information hiding, and architectural layering.
* **False positive:** A deliberate cross-cutting migration such as introducing structured logging, distributed tracing, or replacing a serialization library across the entire system.

### Circular Dependency Appears

* **What you actually observe:** Two modules that previously built independently now require each other, or the build system reports an import cycle.
* **What it's usually a symptom of:** Responsibilities have drifted until ownership is no longer clear.
* **Where the real treatment lives:** The chapters on dependency direction, layering, and abstraction boundaries.
* **False positive:** A temporary cycle during a carefully staged refactor that is removed before merge.

### Every Service Knows Every Other Service

* **What you actually observe:** Configuration files, client libraries, or deployment manifests contain direct connections to most other services in the system.
* **What it's usually a symptom of:** A distributed monolith with excessive runtime coupling.
* **Where the real treatment lives:** The chapters on service boundaries, dependency management, and communication patterns.
* **False positive:** An API gateway, service mesh control plane, or observability collector that is intentionally aware of the entire topology.

### Configuration Change Requires Code Changes

* **What you actually observe:** Adding a new region, customer, or deployment environment requires editing application logic instead of configuration.
* **What it's usually a symptom of:** Configuration and behavior have become intertwined.
* **Where the real treatment lives:** The chapters on configuration management and separation of policy from mechanism.
* **False positive:** Systems with intentionally fixed deployments, such as firmware built for a single hardware target.

---

## 2. API Design Smells

### Boolean Parameter Explosion

* **What you actually observe:** Public methods accumulate parameters like `includeInactive`, `force`, `skipValidation`, `dryRun`, and `ignoreCache`.
* **What it's usually a symptom of:** Multiple unrelated behaviors have been collapsed into one interface.
* **Where the real treatment lives:** The chapters on API design, cohesive interfaces, and explicit modeling.
* **False positive:** Thin wrappers over operating-system or protocol APIs whose options directly mirror an external specification.

### Every Caller Immediately Converts the Return Type

* **What you actually observe:** Most call sites immediately map, unwrap, copy, or translate the same API result into another representation.
* **What it's usually a symptom of:** The API returns the wrong abstraction for its consumers.
* **Where the real treatment lives:** The chapters on interface design and ownership boundaries.
* **False positive:** Translation at an intentional boundary such as converting database models into externally exposed DTOs.

### Version Numbers in Method Names

* **What you actually observe:** Methods such as `CreateUserV2`, `GetOrdersV3`, or `ProcessNew2`.
* **What it's usually a symptom of:** Compatibility strategy has been replaced with API duplication.
* **Where the real treatment lives:** The chapters on compatibility, API evolution, and deprecation.
* **False positive:** Generated client libraries for externally versioned protocols.

### One Endpoint Returns Different Shapes

* **What you actually observe:** A REST endpoint or RPC method returns structurally different payloads depending on flags or hidden server behavior.
* **What it's usually a symptom of:** The interface no longer represents a single contract.
* **Where the real treatment lives:** The chapters on API contracts and versioning.
* **False positive:** GraphQL queries, where the client explicitly controls the response shape.

---

## 3. Code Organization Smells

### Utility Folder Growth

* **What you actually observe:** `Utils`, `Helpers`, or `Common` becomes one of the largest directories in the repository.
* **What it's usually a symptom of:** Ownership has dissolved and unrelated responsibilities have accumulated.
* **Where the real treatment lives:** The chapters on cohesion, module organization, and naming.
* **False positive:** A language runtime or standard library implementation intentionally collecting broadly reusable primitives.

### Files That Never Stop Growing

* **What you actually observe:** Individual source files exceed thousands of lines while continuing to receive unrelated changes.
* **What it's usually a symptom of:** Multiple responsibilities have accumulated behind one compilation unit.
* **Where the real treatment lives:** The chapters on cohesion and code organization.
* **False positive:** Generated source files that are never edited manually.

### One Class Has Hundreds of Imports

* **What you actually observe:** A single class depends directly on dozens of unrelated modules.
* **What it's usually a symptom of:** The class has become an orchestrator, coordinator, validator, formatter, and persistence layer simultaneously.
* **Where the real treatment lives:** The chapters on responsibility boundaries and dependency management.
* **False positive:** Composition roots or dependency injection bootstrap code.

### Copy-Paste Edits Across Many Files

* **What you actually observe:** The same logical change must be repeated manually across numerous files with only minor differences.
* **What it's usually a symptom of:** Missing abstraction or duplicated ownership.
* **Where the real treatment lives:** The chapters on abstraction, duplication, and the indirection tax.
* **False positive:** Intentional duplication to preserve isolation between independent domains.

---

## 4. Testing Smells

### Snapshot Approval Becomes the Default

* **What you actually observe:** Most new tests verify large snapshots rather than specific behavior.
* **What it's usually a symptom of:** Tests are optimized for convenience rather than intent.
* **Where the real treatment lives:** The chapters on test design and meaningful assertions.
* **False positive:** UI rendering tests where structural regressions are the primary concern.

### Tests Retry Until They Pass

* **What you actually observe:** CI reruns failing tests automatically and merges proceed if retries eventually succeed.
* **What it's usually a symptom of:** Flaky tests have been normalized instead of investigated.
* **Where the real treatment lives:** The chapters on determinism, test reliability, and CI.
* **False positive:** Infrastructure outages outside the team's control, such as transient cloud provider failures.

### Coverage Increases While Bugs Continue

* **What you actually observe:** Coverage numbers rise, but escaped defects do not decrease.
* **What it's usually a symptom of:** The team is optimizing coverage instead of fault detection.
* **Where the real treatment lives:** The chapter on coverage and what it actually measures.
* **False positive:** Early phases of building a regression suite after years without automated tests.

### Every Test Uses the Entire Stack

* **What you actually observe:** Even simple business-rule tests require databases, queues, containers, or full application startup.
* **What it's usually a symptom of:** Poor separation between logic and infrastructure.
* **Where the real treatment lives:** The chapters on the testing pyramid, dependency inversion, and architectural boundaries.
* **False positive:** A dedicated end-to-end test suite validating production deployment paths.

---

## 5. Engineering Process Smells

### Every Issue Is High Priority

* **What you actually observe:** Sprint boards, issue trackers, or incident queues contain mostly "critical" or "urgent" labels.
* **What it's usually a symptom of:** Prioritization has stopped distinguishing work.
* **Where the real treatment lives:** The chapters on prioritization, engineering trade-offs, and process overhead.
* **False positive:** An active production incident affecting core business functionality.

### Design Reviews Happen After Implementation

* **What you actually observe:** Major architectural discussions begin only after thousands of lines of code are already written.
* **What it's usually a symptom of:** Design decisions are being validated after implementation rather than before.
* **Where the real treatment lives:** The chapters on design reviews and architectural decision-making.
* **False positive:** Exploratory prototypes whose purpose is to discover unknown constraints.

### Long-Lived Feature Branches

* **What you actually observe:** Branches remain open for weeks or months while diverging substantially from main.
* **What it's usually a symptom of:** Integration risk is accumulating.
* **Where the real treatment lives:** The chapters on branching strategy and continuous integration.
* **False positive:** Large-scale repository migrations isolated until a coordinated cutover.

### Same Incident Happens Repeatedly

* **What you actually observe:** Incident timelines repeatedly reference earlier tickets with nearly identical failure modes.
* **What it's usually a symptom of:** Root causes are not being eliminated.
* **Where the real treatment lives:** The chapters on postmortems, operational improvement, and process feedback loops.
* **False positive:** Independent failures caused by recurring third-party outages outside the team's control.

---

## 6. Git and Delivery Smells

### Merge Conflict Every Week

* **What you actually observe:** The same files generate merge conflicts across unrelated features.
* **What it's usually a symptom of:** Shared ownership bottlenecks or poorly separated responsibilities.
* **Where the real treatment lives:** The chapters on code organization and branching strategy.
* **False positive:** Central dependency manifests or version files that naturally change in nearly every branch.

### Release Checklist Keeps Growing

* **What you actually observe:** Production releases require dozens of manual verification steps.
* **What it's usually a symptom of:** Automation has failed to replace repetitive operational work.
* **Where the real treatment lives:** The chapters on deployment automation and engineering process.
* **False positive:** Safety-critical industries with regulatory sign-off requirements.

### Emergency Commits Directly to Main

* **What you actually observe:** Production fixes routinely bypass normal review and validation.
* **What it's usually a symptom of:** Delivery processes cannot respond fast enough under pressure.
* **Where the real treatment lives:** The chapters on deployment strategy and release engineering.
* **False positive:** Controlled incident response during an active outage with documented follow-up review.

### Frequent Rollbacks

* **What you actually observe:** Release dashboards show a significant fraction of deployments being immediately reverted.
* **What it's usually a symptom of:** Validation before deployment is insufficient.
* **Where the real treatment lives:** The chapters on testing, deployment strategies, and progressive delivery.
* **False positive:** Chaos engineering exercises intentionally validating rollback procedures.

---

## 7. Documentation Smells

### README Starts With "TODO"

* **What you actually observe:** The primary documentation page is largely incomplete months after the project began.
* **What it's usually a symptom of:** Documentation ownership is undefined.
* **Where the real treatment lives:** The chapters on documentation strategy and ownership.
* **False positive:** Internal prototypes not intended for reuse.

### Architecture Diagram Doesn't Match the Repository

* **What you actually observe:** Components shown in documentation no longer exist or are named differently from the code.
* **What it's usually a symptom of:** Documentation has stopped evolving with the system.
* **Where the real treatment lives:** The chapters on living documentation.
* **False positive:** Historical architecture documentation explicitly preserved for retrospective analysis.

### On-Call Runbook Begins With Investigation

* **What you actually observe:** The first instruction in an incident guide is effectively "figure out what's wrong."
* **What it's usually a symptom of:** The document records knowledge instead of operational procedure.
* **Where the real treatment lives:** The chapter on runbooks and operational documentation.
* **False positive:** Brand-new incident classes where investigation is genuinely the first required step.

### Wiki Search Returns Multiple Conflicting Answers

* **What you actually observe:** Different pages describe different deployment processes, APIs, or operational procedures.
* **What it's usually a symptom of:** There is no authoritative source of truth.
* **Where the real treatment lives:** The chapters on documentation ownership and maintenance.
* **False positive:** Documentation intentionally covering multiple supported product versions.

---

## 8. Observability Smells

### Every Log Says "Starting"

* **What you actually observe:** Logs contain many lifecycle messages but almost no business identifiers, outcomes, or contextual fields.
* **What it's usually a symptom of:** Logging optimized for developers rather than operators.
* **Where the real treatment lives:** The chapters on structured logging and observability.
* **False positive:** Very early bootstrap code before request context exists.

### Incident Begins With Adding Logging

* **What you actually observe:** The first debugging step during production incidents is deploying extra instrumentation.
* **What it's usually a symptom of:** Observability requirements were deferred until failure occurred.
* **Where the real treatment lives:** The chapters on logging, tracing, and metrics.
* **False positive:** Investigating a previously unknown failure mode.

### Dashboard Always Green During Outages

* **What you actually observe:** Customers report failures while operational dashboards show healthy systems.
* **What it's usually a symptom of:** Metrics measure infrastructure instead of user experience.
* **Where the real treatment lives:** The chapters on SLOs, SLIs, and observability.
* **False positive:** Partial regional failures not yet covered by monitoring rollout.

### Correlation IDs Disappear Mid-Request

* **What you actually observe:** Distributed traces terminate unexpectedly when crossing service boundaries.
* **What it's usually a symptom of:** Context propagation is incomplete.
* **Where the real treatment lives:** The chapters on distributed tracing and request propagation.
* **False positive:** Intentional asynchronous workflows that begin a separate trace.

---

## 9. Concurrency Smells

### Global Lock Around Everything

* **What you actually observe:** Profiling shows most worker threads blocked on one mutex.
* **What it's usually a symptom of:** Synchronization has replaced concurrency.
* **Where the real treatment lives:** The chapters on synchronization strategy and contention.
* **False positive:** Protecting a genuinely singleton hardware device or operating-system resource.

### Sleep Added to Fix Race Condition

* **What you actually observe:** Production code contains delays intended to "give another thread time."
* **What it's usually a symptom of:** Timing has replaced synchronization.
* **Where the real treatment lives:** The chapters on concurrency correctness.
* **False positive:** Intentional rate limiting or exponential backoff for external systems.

### Queue Length Only Grows

* **What you actually observe:** Message queues accumulate work faster than consumers remove it during normal load.
* **What it's usually a symptom of:** Sustained throughput imbalance or missing backpressure.
* **Where the real treatment lives:** The chapters on queues, backpressure, and capacity planning.
* **False positive:** Predictable burst processing such as overnight batch ingestion.

### Shared Mutable Cache Without Ownership

* **What you actually observe:** Multiple threads update the same in-memory cache through unrelated code paths.
* **What it's usually a symptom of:** Shared state has no clear synchronization strategy.
* **Where the real treatment lives:** The chapters on shared state versus message passing and ownership.
* **False positive:** Lock-free data structures whose correctness has been deliberately designed and verified.

---

## 10. Security Smells

### Secrets in Source Control

* **What you actually observe:** API keys, passwords, or private certificates appear in repository history.
* **What it's usually a symptom of:** Secret management has been bypassed.
* **Where the real treatment lives:** The chapters on secrets management and secure deployment.
* **False positive:** Test credentials intentionally generated for public examples and documented as non-sensitive.

### Permissions Keep Expanding

* **What you actually observe:** Service accounts continually gain additional permissions while rarely losing any.
* **What it's usually a symptom of:** Least privilege is no longer being maintained.
* **Where the real treatment lives:** The chapters on authentication, authorization, and operational security.
* **False positive:** Initial platform bootstrap before responsibilities have been partitioned.

### Dependency Upgrade Deferred Indefinitely

* **What you actually observe:** Security advisories remain open release after release because upgrades are considered too risky.
* **What it's usually a symptom of:** Dependency management has become brittle.
* **Where the real treatment lives:** The chapters on dependency management and supply-chain risk.
* **False positive:** Deliberately waiting for a vendor patch after a regression in the first fixed release.

### Security Checks Disabled in CI

* **What you actually observe:** Static analysis or vulnerability scanning jobs are routinely skipped to make builds pass.
* **What it's usually a symptom of:** Delivery speed is overriding risk management.
* **Where the real treatment lives:** The chapters on CI, security automation, and engineering process.
* **False positive:** Temporary disablement while repairing a broken scanning tool, with compensating review in place.

---

## 11. Performance Smells

### Database Query Count Grows With Result Size

* **What you actually observe:** Loading 100 records performs roughly 100 additional queries.
* **What it's usually a symptom of:** The N+1 query problem.
* **Where the real treatment lives:** The chapters on data access patterns and performance.
* **False positive:** Administrative tools intentionally loading a handful of records where simplicity outweighs optimization.

### CPU Mostly Idle but Latency High

* **What you actually observe:** Systems spend little time on CPU while request latency remains poor.
* **What it's usually a symptom of:** Waiting on I/O, lock contention, or external dependencies rather than computation.
* **Where the real treatment lives:** The chapters on performance analysis, concurrency, and tail latency.
* **False positive:** Systems intentionally throttled to comply with external rate limits.

### Cache Hit Rate Falls During Traffic Spike

* **What you actually observe:** Cache dashboards show hit rates collapsing just as request volume increases.
* **What it's usually a symptom of:** Cache stampede or ineffective cache population strategy.
* **Where the real treatment lives:** The chapters on caching strategies and cache stampede prevention.
* **False positive:** Planned cache invalidation after a major deployment or data migration.

### Performance Optimizations Appear Without Measurements

* **What you actually observe:** Pull requests introduce complex optimizations but contain no benchmark, profile, or production measurement.
* **What it's usually a symptom of:** Optimization targets are assumed rather than measured.
* **Where the real treatment lives:** The opening chapters on optimization targets and the performance engineering chapters on profiling and measurement.
* **False positive:** Emergency mitigation for an actively measured production bottleneck where formal benchmarking follows after the incident.
