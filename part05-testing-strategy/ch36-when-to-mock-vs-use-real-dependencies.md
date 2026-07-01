# Ch 36 — When to Mock vs. Use Real Dependencies

**Prerequisites:** [The Testing Pyramid](ch34-the-testing-pyramid.md), [What Belongs at Each Layer](ch35-what-belongs-at-each-layer.md), [Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Layered, Hexagonal, and Ports-and-Adapters Architecture](../part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md)

**New vocabulary introduced:** test double, dummy, stub, spy, mock, fake, refactoring fragility, state verification, interaction verification

**Key takeaways:**
- The test double taxonomy — dummy, stub, spy, mock, fake — has precise meanings defined by Fowler's "Mocks Aren't Stubs" (2004). Using "mock" as a catch-all conflates four different testing techniques and obscures the intent of a test.
- Mocks verify interactions: that collaborators were called in the right order with the right arguments. Fakes and real objects verify state: that the system produced the correct output. These are not equivalent forms of confidence.
- The classicist (Detroit) school uses real collaborators wherever possible; the mockist (London) school replaces all collaborators with mocks. The classicist approach produces tests that survive internal refactoring; the mockist approach ties test suites to implementation structure, not behavior.
- Prefer real or near-real dependencies where the cost is acceptable. Replace a dependency only when it is outside your control, introduces non-determinism you cannot remove, or is prohibitively expensive to run.
- Refactoring fragility — a test suite that breaks during valid internal restructuring because it specifies implementation rather than behavior — is the dominant cost of over-mocking.

---

[Ch 35](ch35-what-belongs-at-each-layer.md) established which layer should test a given piece of code. This chapter answers the question within a layer: when a unit test needs a collaborator, should that collaborator be real or replaced?

This question has a taxonomy, two opposing philosophical traditions, and a practical position. The taxonomy must come first, because the traditions are describing different objects under the same name.

---

### The Test Double Taxonomy

Martin Fowler's 2004 essay *"Mocks Aren't Stubs"* formalized five categories of test substitute, collectively called **test doubles**. Engineering teams that use "mock" for all five accumulate real communication failures: "mock the repository" could reasonably describe four distinct techniques with different implications.

**Dummy** — An object passed into a method signature to satisfy the compiler but never invoked. A dummy configuration struct passed to a constructor that ignores it in the path under test.

**Stub** — A component that returns predetermined responses to specific calls. It provides data the code under test requires; it does not track how or whether it was called, and the test makes no assertions about it directly.

**Spy** — An enhanced stub that records its interactions — how many times a method was called, with what arguments — for inspection after the fact. The test examines what the spy captured once execution completes.

**Mock** — A substitute pre-programmed with behavioral expectations before execution. If the code under test fails to call the right methods in the right order with the right arguments, the mock fails the test immediately. Mocks verify interactions, not outputs.

**Fake** — A working implementation with a simplified backing mechanism unsuitable for production: an in-memory repository that stores records in a hash map, a local SMTP server that records messages in memory. A fake implements the same interface as the production dependency and performs real work; it simply takes shortcuts a deployed system cannot.

The operationally significant division is between doubles that verify **state** (the system produced the correct output — stubs, fakes, real collaborators) and doubles that verify **interactions** (the system called its collaborators correctly — mocks, and sometimes spies). The choice between these two approaches is the central decision this chapter makes.

---

### Verify Behavior, Not Interactions

**What it is:** The choice between tests that assert what the system *produces* (state verification) and tests that assert what the system *calls* (interaction verification).

**Why it exists:** A test using real collaborators, fakes, or stubs verifies that assembled code produces the correct observable outcome — the right value was returned, the right record was written, the right message was generated. A test using mocks verifies that the code invoked specific methods in a specific sequence. These answer different questions. Behavior can be correct while implementation structure changes; interaction specifications can pass while behavior breaks. An engineer who changes an internal call sequence for performance reasons and causes a hundred mock-based tests to fail has not broken the software — the software works the same way. But the tests will not say so.

**Options:**

1. **State verification** — test with real collaborators, in-memory fakes, or stubs; assert the final output or persisted state.
2. **Interaction verification** — test with mocks that declare what methods must be called; assert those expectations were met.

**Trade-offs:**

[Strong Recommendation] **State verification as the default.** Tests that verify observable behavior survive internal restructuring: if an engineer splits a method, merges two helpers, or replaces sequential calls with a batched equivalent, a state-verifying test is unaffected as long as the output is correct. The behavior was never part of the specification; the behavior was the specification.

**Interaction verification is appropriate for outbound side effects** — behaviors that produce no direct return value the test can inspect: dispatching an asynchronous notification, publishing an event to a message broker, writing to an audit log. There is no output to assert against; the call itself is the behavior. A spy that captures the outgoing notification payload or a mock that asserts a publish method was called once is the natural tool in this narrow situation.

**When to choose:** Default to state verification. Use interaction verification only when the side effect itself is the behavior under test and no observable state change exists to assert against.

**Common failure modes:**

*The Brittle Expectation Lock.* A billing engine test uses a mock that expects `fetchDiscountRate()` to be called exactly twice with specific account IDs. A later optimization combines both calls into a single batched lookup. The final invoice total is identical. The mock framework fails the test anyway because it was verifying internal call structure, not behavior. Correcting the test requires rewriting test code to accommodate a performance change that altered nothing observable.

**Example:** A billing engine test verifies an invoice total using a *stub* that returns a fixed discount rate whenever queried. The test passes the stub a fixed `PremiumCustomer` value and asserts the computed total. If the engine later caches discount lookups internally, the test is unaffected — it never specified how many times the stub would be called, only what the engine should produce. To additionally verify that a success notification is dispatched, the same test injects a *spy* that records the outgoing email payload; after execution, the test inspects what the spy captured. State verification handles the computation; a spy handles the side effect. Each tool applies to the situation it was designed for.

---

### Classicist or Mockist Testing Strategy

**What it is:** Two schools of thought about what "unit" in unit testing means, and therefore how extensively test doubles should replace collaborators.

**Why it exists:** The *classicist* school (associated with Kent Beck and the Detroit tradition) defines a unit as a cohesive piece of behavior. Tests exercise real collaborators freely as long as they operate in memory without I/O. Test doubles are reserved for external systems, non-deterministic resources, and infrastructure. The *mockist* school (associated with the London tradition) defines a unit as a single class isolated from all collaborators. Every dependency is replaced with a mock, and the test asserts on call patterns and argument structures.

**Options:**

1. **Classicist (Detroit) strategy** — use real collaborators for all components within the system boundary; replace only external systems, non-deterministic resources, and infrastructure.
2. **Mockist (London) strategy** — replace all collaborators with mocks regardless of whether they are internal or external; isolate the single class or file under test completely.

**Trade-offs:**

[Strong Recommendation] **Classicist strategy as the default.** The dominant cost of the mockist approach is *refactoring fragility*: a test suite built on interaction specifications becomes a rigid transcript of the code's current internal structure. Every refactor — extracting a helper method, reorganizing collaborators, combining calls — invalidates expectations that describe no behavioral change. Engineers who find that their test suite punishes refactoring stop refactoring; the accidental complexity that tests were supposed to help control instead accumulates unchecked.

The classicist approach incurs a different cost: when a shared collaborator contains a bug, tests that exercise it transitively fail. This failure pattern is diagnosable and visible — a cluster of related failures indicates a problem in a shared component. The mockist failure mode — tests passing while production breaks because collaborators were replaced with assumptions — is the more dangerous direction.

**Mockist techniques have legitimate value** for components whose primary responsibility is coordination: a workflow orchestrator, a message dispatcher, an inter-service gateway. When the sequence and target of calls *is* the behavior — not a byproduct of it — interaction verification is appropriate.

**When to choose:** Default to classicist. Apply mockist techniques selectively for high-level coordinators whose correctness is defined by *how* they route collaborators, not by the values they compute.

**Common failure modes:**

*The Green Mirage.* A team achieves high coverage using mocks for every collaborator. Tests run in seconds and pass reliably in CI. In production, the application crashes immediately: the mocks were configured to return values that the real dependencies cannot produce. Because every collaborator was replaced, the tests verified the assumptions embedded in the mocks — they never verified the actual behavior of the system. The test suite's green signal was measuring compliance with the team's own assumptions, not with reality.

**Example:** A user registration service invokes a password hasher, a database repository, and a notification publisher. A mockist test replaces all three with mocks and asserts that `hash()`, then `save()`, then `notify()` were called in order. A classicist test uses the real password hasher (fast, deterministic, in-memory), an in-memory repository fake implementing the same interface, and a notification spy. It asserts that a validly hashed record was actually committed to the state store and that one notification was captured. If the engineer later extracts the hashing step into a standalone service, the classicist test requires no changes — the behavior is the same. The mockist test fails because the call sequence changed.

---

### Prefer Real or Near-Real Dependencies

**What it is:** The decision about whether a dependency should be replaced at all — and when a real or near-real instance should be used instead of any substitute.

**Why it exists:** Every test double introduces an assumption about how the production dependency behaves. Those assumptions can drift silently. The production system changes; the mock does not. Tests continue to pass; the failure waits for deployment. Preferring real dependencies where possible is not a purist position — it is an acknowledgment that a substitute without the production dependency's actual behavior is not verifying it.

**Options:**

1. **Real dependency** — the production implementation, run in a container or in-process where applicable.
2. **In-memory fake** — the same interface with a simplified backing mechanism: a map-backed repository, a local SMTP implementation.
3. **Stub or mock** — canned responses or pre-programmed interaction expectations.

**Trade-offs:**

[Strong Recommendation] **Prefer real or near-real dependencies where the operational cost is acceptable.** The historical case for mocking infrastructure — spawning a database is slow, fragile, and environment-dependent — has been substantially undermined by container tooling. Testcontainers (available for Java, Go, Python, .NET, and others) starts a production-identical PostgreSQL, Redis, or Kafka instance in seconds within the test run and tears it down on completion. The cost of "too expensive to run real" has moved significantly downward.

**In-memory fakes are appropriate** at the unit layer, where domain logic exercises repository behavior at high speed without I/O overhead. A fake must genuinely implement the same interface and behavioral contract as the real adapter — a fake is not a mock with simpler internals; it is a working implementation with a lighter backing store.

**Stubs and mocks are the right tool** when the dependency is:
- **External to your control** — a third-party payment processor, an external SaaS API, a licensed service you cannot provision locally.
- **Irreducibly non-deterministic** — the system clock, a random number generator, network latency, hardware sensors.
- **Prohibitively expensive or inaccessible** — a legacy mainframe, a heavily rate-limited external enterprise system, a licensed data provider.

Outside these three categories, "the dependency is complicated to set up" is not a sufficient reason to mock it. Complicated setup is a signal to invest in Testcontainers or a shared test fixture, not to substitute the dependency with assumptions.

**When to choose:** If the dependency belongs to your system, starts within a few seconds, and runs deterministically, use it — either directly or in a container. If it is external, non-deterministic, or genuinely impractical to run, replace it with the least artificial substitute available: a fake before a stub, a stub before a mock.

**Common failure modes:**

*Dialect Drift.* A team tests its PostgreSQL repository against an in-memory SQLite substitute because SQLite is lighter and requires no daemon. An engineer writes a reporting query using PostgreSQL window functions. SQLite accepts the query without complaint — it doesn't implement those functions but fails silently rather than raising a syntax error in the test harness. Tests pass. Production crashes with a fatal SQL exception. The integration test was validating the stub's silence, not the database's execution.

**Example:** An order fulfillment system uses a two-tier strategy. Domain logic is tested with an in-memory repository fake that stores records in a thread-safe map — millisecond execution, thousands of business-rule variations per second. The persistence adapter layer is tested against a real, ephemeral PostgreSQL instance via Testcontainers: every INSERT, index constraint, foreign key rule, and migration script is exercised against the actual engine. No mock touches the database layer. The in-memory fake serves the unit layer; the real container serves the integration layer. Each replacement exists because it removes a genuine obstacle, not because a mocking framework made replacement easy.

---

### Why Smart Engineers Disagree: Design Tool or Safety Net?

The foundational disagreement is about what a test suite is *for*.

The mockist argument is that a test suite is a design tool. When every collaborator must be explicitly mocked, the mock configuration surface becomes a live coupling measurement: a class that requires eight mocks to initialize has eight dependencies, and that number is itself a feedback signal. Mocks force interface clarity upfront because you cannot mock a dependency you haven't made explicit. In this view, classicist testing masks poor design — a cohesion-poor, tightly coupled module can pass all its tests if the global output happens to be correct, and the design problem goes unnoticed.

The classicist argument is that a test suite is a safety net. A safety net that fails every time an engineer restructures code — even when the behavior is unchanged — trains engineers to avoid restructuring. The result is that the very tool meant to enable refactoring actively discourages it. A test that specified interactions but not outcomes was never verifying correctness; it was verifying a particular execution path and calling it a test.

The practical resolution is architectural. Hexagonal architecture from [Ch 11](../part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md) makes port interfaces explicit by construction — the design-feedback benefit the mockist school seeks from mocks arrives structurally from the architecture, without the refactoring cost. Domain logic in the core has no hidden dependencies because the structure enforces isolation. When the architecture already surfaces coupling, mocks are not needed to expose it.
