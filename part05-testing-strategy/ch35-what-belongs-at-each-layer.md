# Ch 35 — What Belongs at Each Layer

**Prerequisites:** [The Testing Pyramid](ch34-the-testing-pyramid.md), [Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Layered, Hexagonal, and Ports-and-Adapters Architecture](../part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md), [When to Split Files vs. Keep Together](../part04-code-organization/ch29-when-to-split-files-vs-keep-together.md)

**New vocabulary introduced:** architectural seam, testability diagnostic

**Key takeaways:**
- Unit tests belong where behavior is purely computational: pure functions, domain logic, validation, data transformations, state machines. If adding a database or network call contributes nothing to verifying the behavior, it should not be present.
- Integration tests belong at architectural boundaries: database queries, ORM mappings, HTTP adapters, message serializers. Test these against real or near-real infrastructure — an in-memory substitute that doesn't share the production engine's dialect will pass tests and allow production to fail.
- End-to-end tests belong at complete user journeys: the paths where the assembled behavior of all components together is what is being validated. A critical handful, not a comprehensive catalog.
- Difficulty writing a unit test is a design signal, not a testing problem. Code that is hard to isolate is code that mixes concerns that should be separated.
- Test co-location — tests living next to the code they verify — is the natural expression of package-by-feature organization. It prevents the structural drift that parallel test trees accumulate over time.

---

[Ch 34](ch34-the-testing-pyramid.md) established that different tests answer different questions at different costs. This chapter answers the practical question that follows: for any given piece of code, which layer should test it?

This is a design question as much as a testing question. The layer that naturally fits a piece of code reveals what kind of responsibility that code has. If determining the right layer is difficult, the design is often the real problem — the code is entangling responsibilities that belong at different levels of the architecture.

---

### Unit-Test Pure Behavior

**What it is:** The constraint that unit tests cover behavior whose correctness depends only on inputs, logic, and outputs — with no external I/O, no clock, no filesystem, no network, no database.

**Why it exists:** Pure behavior is deterministic: the same inputs always produce the same outputs, regardless of environment, system time, or local file layout. This *determinism* is what makes unit tests cheap and precise. They execute in milliseconds, run identically on every machine, and when they fail they point directly at the code path that is broken. Adding unnecessary infrastructure to a unit test trades away that precision for nothing.

**Options:**

1. **Pure-domain isolation** — test business logic using only in-memory inputs and outputs, with no framework, network, or persistence dependencies.
2. **Framework-entangled isolation** — test business logic inside framework-provided test contexts that manage object lifetimes, configuration, or persistence.

**Trade-offs:**

[Strong Recommendation] **Pure-domain isolation** for all core business logic. A pricing engine, a discount calculator, a state machine, a validation rule, a data transformer — these exist independently of how their inputs arrive or where their outputs go. Testing them without infrastructure is not a limitation; it is the natural expression of what they are. A domain module tested this way can survive framework upgrades, database migrations, and infrastructure changes without touching a single test line.

**Framework-entangled isolation** accelerates early development by allowing validation rules to live directly inside framework models. The cost arrives at scale: every test bootstraps the framework, turning a millisecond assertion into a multi-second setup operation. The unit base of the pyramid becomes too slow to run continuously.

**When to choose:** Unit tests for any behavior whose correctness depends solely on computation. If the test would pass or fail identically whether or not a database was present, the database should not be present.

**Common failure modes:**

*The Hidden I/O Leak.* An engineer writes a unit test for a validation function. The function internally reads a configuration file to check a locale setting, or calls a library that queries the system timezone. The test passes locally. In a CI container without that configuration or in a different timezone, it fails. The test was never isolated; the I/O dependency was simply invisible.

**Example:** In a hexagonal architecture, an international billing engine computes compounded regional VAT from raw domain values. Its unit tests pass floats and custom domain types directly into the computation and assert the result. The test file imports no SQL, no HTTP, no framework context. The test suite for a complex tax calculation engine can run thousands of cases in under a second precisely because it contains nothing that doesn't belong there.

---

### Integration-Test Boundaries

**What it is:** The decision to test the interaction between a component and the external system it communicates with — verifying that the communication itself is correct, not just that the component's internal logic is correct.

**Why it exists:** Two independently correct components can fail when they interact. A repository may contain perfectly valid SQL that maps columns incorrectly. An HTTP client may serialize requests correctly by its own logic but violate the target service's contract. An ORM mapping may look right until a migration renames a column. Unit tests cannot detect any of these failures because they occur at the boundary, not inside either component. The *architectural seam* — the interface where one component touches another — is exactly what the integration layer tests.

**Options:**

1. **Real infrastructure integration** — test adapters against actual instances of the dependency: a real PostgreSQL database, a real Redis cache, a real Kafka broker, spun up in a container.
2. **In-memory emulation** — test adapters against a lightweight substitute that approximates the real dependency, such as an embedded H2 database in place of PostgreSQL.

**Trade-offs:**

[Strong Recommendation] **Real infrastructure integration** for all production database adapters, message brokers, and any operation that uses vendor-specific behavior. The specific reason in-memory emulation fails is *dialect divergence*: SQL that runs cleanly against an H2 in-memory engine may use syntax or locking behavior that triggers an error on PostgreSQL. The in-memory substitute accepts the invalid query without complaint; production crashes. The integration test that was supposed to catch this failure instead provided false confidence. Testcontainers (Java, Go, Python, and others) has shifted the cost threshold: spinning up a real PostgreSQL instance in a Docker container during CI is now cheap enough that the tradeoff for most teams no longer favors the substitute.

**In-memory emulation** is acceptable when the data access model is so simple — standard key-value operations, basic collection semantics — that no dialect divergence exists in practice. This is a narrower category than it appears.

**When to choose:** Whenever the correctness of an adapter depends on communicating with another component, test that communication with a real or near-real instance. The boundary is what is being verified; a substitute that doesn't share the boundary's actual behavior isn't verifying it.

**Common failure modes:**

*The Mocked Database Illusion.* A team tests its database adapter layer by mocking the database client. Tests run fast and pass in CI. In production, the application crashes with SQL syntax exceptions — a column was renamed in a migration script, but the mock accepted the stale column name without objection. The test passed because the mock was told what to return; it verified nothing about the actual database.

**Example:** A billing module's database adapter writes financial records through a port interface. Its integration tests spin up a production-identical PostgreSQL instance via Testcontainers, execute the real INSERT query, read the persisted state directly, and verify that relational constraints and foreign keys are enforced correctly. The test bypasses mocks entirely because the adapter's correctness is defined by what the database actually does with its queries.

---

### End-to-End Test Complete User Journeys

**What it is:** The small, curated set of tests that exercise the entire assembled application — from the external interface through all services and infrastructure — to validate the behaviors that only emerge when everything is deployed and running together.

**Why it exists:** Some failures are invisible until all components are assembled: a misconfigured environment variable, a broken API gateway route, a CDN caching rule that intercepts authenticated requests, a deployment that assembled the correct components in the wrong order. Unit and integration tests cannot catch these because they do not assemble the full system. A small E2E layer exists to provide a macro-level smoke check over the complete operational topology.

**Options:**

1. **Critical journey guardrails** — a small, high-value set of E2E scripts covering the paths that define the system's core capability.
2. **Comprehensive boundary regression** — E2E tests that attempt to cover every feature, variation, and edge case at the full-stack level.

**Trade-offs:**

[Strong Recommendation] **Critical journey guardrails only.** E2E tests cover the journeys where the assembled behavior is itself what is being validated. Every other verification belongs lower in the pyramid. The consequence of comprehensive boundary regression is the ice cream cone: an unmaintainable, slow, flaky suite that eventually becomes too unreliable to trust.

**When to choose:** End-to-end tests for complete user journeys that cross multiple architectural boundaries and require the assembled system to succeed: user registration, checkout, authentication, payment processing, document upload. Not for individual business rules, discount calculations, input validations, or error messages — those belong at the unit or integration layer regardless of the fact that users eventually encounter them through the UI.

**Common failure modes:**

*The Brittle DOM Selector Blockade.* An engineer writes an E2E test to verify that an error message appears when an invalid coupon code is entered. A designer later moves the error component to a side modal and updates a CSS class name for layout reasons. The underlying validation logic, API, and database are unchanged and correct. The E2E test fails because its DOM selector no longer matches the element. A critical deployment is blocked for hours while someone finds and updates a locator string. The test was testing the wrong thing at the wrong layer.

**Example:** A distributed e-commerce platform restricts its E2E suite to three scripts: `verifyUserCanRegister()`, `verifyUserCanSearchAndCart()`, and `verifyUserCanCompleteCheckout()`. Each asserts only a broad success state — reaching the `/receipt` route, receiving a confirmation email. Every edge case — expired coupons, invalid quantities, decimal rounding — is pushed down to unit tests in the domain layer. The E2E suite stays small, fast, and trustworthy precisely because it does not try to be comprehensive.

---

### Let Testability Reveal Design Quality

**What it is:** The principle that difficulty writing a unit test is a *testability diagnostic* — a signal about the design of the code, not a signal about the difficulty of testing.

**Why it exists:** Well-designed code separates computation from infrastructure interaction. Business logic that depends directly on databases, network calls, the system clock, or a UI framework cannot be isolated for unit testing because there is nothing to isolate from. The test is difficult not because testing is hard but because the responsibilities are entangled. The test is exposing coupling that the design should not have.

**Options:**

1. **Accept the coupling** — push the test up to a higher layer that includes the infrastructure.
2. **Refactor toward explicit boundaries** — separate computation from infrastructure so the business logic can be tested without it.
3. **Skip the test** — accept that this code is untestable at the unit layer.

**Trade-offs:**

[Strong Recommendation] **Refactor toward explicit boundaries** when business logic and infrastructure are entangled. Pushing a test upward postpones the design problem; it does not solve it. The business logic is still coupled to infrastructure, and it will be harder to change, harder to reuse, and harder to reason about. Code with explicit dependencies, clear interfaces, and isolated domain logic naturally becomes easy to test — and that ease is the signal that the design is correct, not just that the tests are convenient.

**When to choose:** If writing a unit test requires constructing a database connection, starting an HTTP server, reading a configuration file, or constructing a significant portion of the application graph — reconsider the design before reconsidering the test layer.

**Example:** Hexagonal architecture from [Ch 11](../part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md) provides the structural answer: domain logic lives in a core that has no knowledge of infrastructure; adapters live at the perimeter and implement the ports the domain defines. The domain is unit-tested without infrastructure; the adapters are integration-tested against real infrastructure. The architecture determines the testing layers; engineers do not have to invent them on a case-by-case basis.

---

### Organize Tests Alongside the Code They Verify

**What it is:** The organizational decision about whether test files live next to the production code they test (co-location) or in a separate parallel directory hierarchy that mirrors the production structure.

**Why it exists:** Tests evolve together with the code they verify. Physical proximity determines whether they actually do. Co-location makes tests discoverable and makes it obvious when a code change might require a test change. A parallel tree requires engineers to maintain a mirrored structure that drifts whenever the production structure changes.

**Options:**

1. **Co-location** — test files live in the same directory as the implementation, named by convention (`user.go` and `user_test.go`, `orders.py` and `test_orders.py`, `Cart.tsx` and `Cart.test.tsx`).
2. **Parallel test tree** — all test files live in a separate top-level directory tree that mirrors production (`src/orders/` matched by `tests/orders/`).

**Trade-offs:**

[Strong Recommendation] **Co-location as the default.** Tests that live next to their implementation are harder to orphan. When a feature module is moved, renamed, or deleted, the co-located tests move with it — or their absence is immediately visible in the same directory. Co-location also reinforces package-by-feature organization ([Ch 27](../part04-code-organization/ch27-file-and-module-structure.md)): the test for an order module is part of the order module, not a separate artifact that happens to reference it.

**Parallel test trees** are appropriate when the language ecosystem or framework makes them the strongly conventional choice (pytest's default discovery model, for example). When that's the case, consistency matters more than the abstract preference for co-location — follow the ecosystem, not the principle in isolation.

**Common failure modes:**

*The Ghost Module Regression.* A team using a parallel test tree refactors the production code, moving and renaming a module. The corresponding test directory is in a separate branch of the file tree and is not noticed during the refactor. The orphaned tests continue to compile against stale import paths for weeks before anyone investigates why they reference a module that no longer exists.

**Example:** Go's toolchain endorses co-location structurally: `_test.go` files are discovered in the same directory as the code they test. Go provides a further refinement — a test file can declare `package shipping` to access unexported identifiers, or `package shipping_test` to test only the public API as a black-box consumer — without leaving the directory. The test type controls access level, not location. Pytest's parallel tree model works effectively when applied consistently; the failure mode is inconsistency, not the approach itself.

---

### Why Smart Engineers Disagree: Public Surface vs. Internal Testing

The persistent debate about unit testing is not whether different code belongs at different layers — almost everyone agrees on that. The disagreement is about whether unit tests should interact exclusively with a component's public API or should access its internal, unexported logic directly.

Engineers who advocate strict public-surface testing argue that unit tests must treat a module as a black box. If an engineer refactors an internal algorithm — splitting a private method, renaming an internal class, restructuring the execution path — while preserving external behavior, no test should break. A test that references private implementation details couples the suite to the code's current shape rather than its behavior. Refactoring becomes dangerous: valid structural improvements that don't change observable behavior still break tests.

Engineers who favor internal testing argue that critical algorithmic logic often lives in private helpers, and that testing only the public surface means testing through significant indirect paths. A complex parsing routine called three levels deep by a public method is easier to verify directly, with targeted inputs, than through the entire public interface. The complexity of the setup required to exercise it through the public API may itself be a maintenance burden.

The practical resolution is architectural. If the internal logic is genuinely complex and independently meaningful, it belongs in its own module with its own public API — and the testing question resolves itself. If it does not warrant a module, it is probably not complex enough to need direct testing beyond what the public surface exercises. Code that is hard to test through its public API is often either too coupled internally or too coarsely defined externally. Both are design signals, not testing problems.
