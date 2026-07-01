# Ch 39 — When Not to Test

**Prerequisites:** [The Testing Pyramid](ch34-the-testing-pyramid.md), [What Belongs at Each Layer](ch35-what-belongs-at-each-layer.md), [When to Mock vs. Use Real Dependencies](ch36-when-to-mock-vs-use-real-dependencies.md), [Fixture-Based Testing](ch37-fixture-based-testing.md)

**New vocabulary introduced:** trust boundary, implementation coupling, negative ROI test

**Key takeaways:**
- Every test has a lifetime cost: it must be written, understood, maintained, debugged, and re-run. A test suite is an engineering system with its own complexity budget, not a free good that only accumulates value as it grows.
- Trivial delegation — a getter, a setter, a one-line pass-through with no conditional logic — provides no meaningful confidence relative to its maintenance cost. If the test is longer or more complex than the code it verifies, the code is probably not where the risk lives.
- Application tests should stop at the *trust boundary*: the line separating your business logic from frameworks, ORMs, and language runtimes that vendors already test extensively. Testing that a framework does what its own test suite already verifies duplicates effort for zero marginal confidence.
- Tests should bind to public behavior, not private implementation. A test that fails when a private helper is renamed or an internal loop is restructured — while the code's externally observable behavior is unchanged — has *implementation coupling*, and it penalizes exactly the refactoring that a healthy test suite should protect.
- Hard-to-test code and not-worth-testing code are different diagnoses requiring different responses. Hard-to-test code signals a design problem — fix the design, don't skip the test. Not-worth-testing code is easy to test but verifies nothing of value — omit the test, don't write it out of habit.
- Deleting a negative-ROI test is a legitimate engineering decision, not a lapse in discipline. A smaller suite that runs fast and is trusted provides more value than a larger suite that is slow and ignored.

---

Every other chapter in this Part argues for testing something. This chapter argues the opposite case: that some code should not be tested, some tests should never be written, and some existing tests should be deleted. The failure mode under discussion is a suite whose maintenance burden exceeds the confidence it actually provides — a cost that accumulates quietly, one low-value test at a time, until the whole system slows down.

A test is not free once it is merged. It is a standing commitment: someone will read it to understand what it verifies, someone will debug it when it fails, and someone will update it every time the surrounding code changes — whether or not that change was a regression. The relevant question for a candidate test is not "could this fail?" but "does verifying it here produce more confidence than it costs to maintain?"

---

### Don't Test Trivial Delegation or Someone Else's Framework

**What it is:** The decision to withhold tests from code with no meaningful branching logic — getters, setters, one-line pass-throughs, simple constructors — and from code whose actual behavior belongs to a framework, ORM, or language runtime rather than to the application.

**Why it exists:** Testing effort should concentrate where failure is plausible and consequential. A getter that returns a value assigned by its setter has no decision-making logic to verify; the only thing that could break it is a defect in the language runtime itself, which is a problem application tests are not equipped to catch and framework vendors have already tested far more thoroughly than any single application team could. The same reasoning applies to dependency injection wiring, ORM-generated SQL, and routing configuration: these are the framework's documented behavior, verified by the framework's own test suite before it ever reached you. An application test asserting that Spring injects a constructor dependency, or that an ORM produces the exact SQL its documentation promises, is re-testing the vendor, not the application.

**Options:**

1. **Test everything uniformly** — every method, every class, every framework interaction gets a dedicated test, regardless of whether it contains meaningful logic.
2. **Test only meaningful behavior** — restrict tests to code containing conditional logic, data transformation, or business rules; leave trivial delegation and framework wiring uncovered, relying on higher-level tests or the framework's own guarantees.

**Trade-offs:**

[Strong Recommendation] **Test only meaningful behavior.** Testing everything uniformly requires no judgment, but it produces large, noisy suites where thousands of tests exist purely to confirm field assignment while genuine business logic gets comparatively less scrutiny per line. Restricting tests to meaningful behavior requires distinguishing trivial code from consequential code — a judgment call with real borderline cases — but it concentrates maintenance effort where it produces return.

**When to choose:** If removing a piece of code would obviously break behavior a user or caller depends on, it is probably worth testing directly. If removing it merely eliminates a pass-through with no decisions of its own, a higher-level test that exercises the code path incidentally usually provides sufficient coverage. For framework interactions, test the code you wrote — how your application behaves once a dependency has been injected, what your repository stores and retrieves — not whether the framework's injection or ORM mechanism itself works as documented.

**Common failure modes:**

*The Tautological Framework Mirror.* An engineer writes a unit test for a plain data object: instantiate it, call `setName("test")`, assert `getName()` returns `"test"`. The only way this test can fail is a catastrophic defect in the language's own memory or string handling — a failure mode the test was never positioned to catch and that would break every other test in the suite simultaneously if it occurred. The test consumes maintenance effort and contributes zero verification value.

**Example:** An engineer adds a test asserting that an ORM generates an exact `SELECT` statement with specific column ordering. When the ORM library is upgraded and reorders columns internally for a performance optimization — a change with no effect on application correctness — the test fails across dozens of modules, and days are spent repairing tests that were verifying the vendor's implementation detail rather than the application's behavior. Testing that the application correctly stores and retrieves data through the repository interface would have caught the same class of real regression without coupling to the ORM's internal SQL generation.

---

### Test Public Behavior, Not Private Implementation

**What it is:** The discipline of binding test assertions exclusively to the externally observable behavior a component exposes to its callers, rather than to its private methods, internal call sequences, or intermediate state.

**Why it exists:** Implementation changes far more frequently than public behavior. A component's private helpers get split, merged, renamed, and reordered constantly during ordinary maintenance and optimization — while its contract with callers stays stable for long periods. A test suite that inspects private state is coupled to *implementation coupling*: it does not describe what the software must do, only how it currently happens to do it. That coupling turns the test suite into a constraint on refactoring rather than a safety net for it. The primary value a test suite is supposed to provide — the freedom to change internal structure fearlessly — is exactly what implementation-coupled tests destroy.

**Options:**

1. **Public-behavior testing** — supply inputs at the module's public boundary and assert only on its observable outputs or state changes.
2. **Private-implementation testing** — use reflection, package-private access, or interaction mocking to assert on internal call sequences, private helper behavior, or intermediate variables.

**Trade-offs:**

[Strong Recommendation] **Public-behavior testing as the default.** It survives internal restructuring: splitting a method, merging two helpers, replacing a loop with a different algorithm — none of it should require touching the test, because none of it changes what the component promises to callers. The cost is that a failure deep inside a large component may require some tracing back from the public assertion to the actual faulty internal path.

**Private-implementation testing** has a narrow legitimate use: isolated, highly non-linear algorithmic engines — a custom tree-balancing routine, a low-level buffer management scheme — where the internal state transition genuinely *is* the essential complexity under test, not an incidental detail of how a simpler goal was achieved.

**When to choose:** If a test would need to be rewritten whenever an internal helper is renamed or an internal sequence is reordered — even though the code's externally observable behavior is completely unchanged — the test is bound to the wrong layer and should be rewritten against the public interface or deleted.

**Common failure modes:**

*The Mock Setup Monolith.* A team tests a service's internal collaboration sequence by mocking every private collaborator and declaring the exact order and arguments each must receive. The resulting test carries forty lines of mock configuration and one line of actual assertion. When an engineer later replaces a sequential internal loop with an equivalent, faster concurrent implementation — with the external API and its results completely unchanged — the mock's ordering expectations fail immediately. The failure has nothing to do with correctness; it is purely a record of exactly how the internals used to work.

**Example:** A payment processing component's public method `ProcessPayments(batch)` remains unchanged in signature and output while its internals are rewritten from a sequential loop to a concurrent map-reduce structure. All production transactions continue processing correctly. A test suite written against the public interface — supplying a batch, asserting on the resulting payment records — passes without modification. A test suite that had been spying on the internal loop's private call sequence fails immediately, not because anything is broken, but because it had frozen the old implementation in place as an unstated part of its specification.

---

### Distinguish Hard-to-Test from Not-Worth-Testing

**What it is:** The judgment separating two categories of code that both resist straightforward test-writing but demand opposite responses: code that is difficult to test because its design is poor, and code that is easy to test but yields no meaningful confidence when tested.

**Why it exists:** These two situations produce a similar surface symptom — engineers hesitating to write a test — but the correct response inverts between them. Code that requires fifteen mocks, deeply nested configuration, or global state initialization just to exercise a single logic path is exhibiting a design symptom: mixed concerns, missing abstraction boundaries, excessive coupling to infrastructure. The correct response is architectural — refactor to isolate the business logic, as the hexagonal seams from [Ch 11](../part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md) describe — not to skip the test or tolerate the complicated one. Code that is trivially easy to test but has no conditional logic of its own — a one-line pass-through, a framework-managed accessor — is exhibiting a different symptom entirely: the test would be simple to write, but it verifies nothing beyond language mechanics. Treating both situations the same way sends effort exactly where it produces the least value: elaborate test scaffolding gets built to work around bad architecture instead of fixing it, while trivial methods accumulate exhaustive coverage simply because they were the path of least resistance.

**Options:**

1. **Refactor difficult code** — when a piece of business logic is hard to test because it entangles domain rules with infrastructure, restructure it so the logic can be isolated and tested cleanly.
2. **Write the complicated test anyway** — accept the elaborate mock setup or deep configuration as a permanent cost of testing entangled code.
3. **Skip tests on trivial code** — when the code has no meaningful branching logic to verify, decide the test is not worth writing at all.

**Trade-offs:**

[Strong Recommendation] **Refactor when code is hard to test; skip when code is not worth testing — and never confuse the two.** Writing a complicated test around entangled code produces immediate verification at the cost of freezing the poor design in place: the mock setup becomes a permanent workaround that the next engineer inherits rather than a temporary accommodation. Skipping a test on genuinely trivial code costs nothing except the discipline to recognize the difference.

**When to choose:** If business logic requires extensive setup because it mixes domain rules with databases, networking, or configuration, treat the difficulty as a design signal and refactor until the logic is independently testable. If a delegating method or trivial accessor has no branching logic of its own, do not write a test simply because writing one would be easy.

**Common failure modes:**

Teams that confuse these two situations end up investing effort in inverse proportion to value: complex mock-laden tests become permanent scaffolding around architecture no one has time to fix, while trivial methods accumulate exhaustive coverage precisely because testing them required no judgment at all.

**Example:** A pricing engine tightly coupled to raw SQL queries is hard to test — every price calculation requires standing up database state to exercise a rule that has nothing to do with persistence. The correct response is to extract the pricing logic behind a port so it can be unit-tested with plain values, not to accept an integration-heavy test suite as the permanent cost of verifying pricing rules. A simple forwarding method that hands a request directly to a repository, with no logic of its own, rarely deserves a dedicated test regardless of how easy one would be to write.

---

### Recognize and Remove Tests with Negative ROI

**What it is:** The ongoing practice of evaluating existing tests against their actual maintenance cost and deleting the ones whose upkeep exceeds the confidence they provide — treating deletion as a legitimate engineering decision rather than an admission of failure.

**Why it exists:** A test suite is not static; it accrues cost throughout the life of the system it verifies. Confidence and maintenance burden do not scale at the same rate — past a certain point, additional tests exist mainly to generate work rather than catch regressions. A suite that takes forty-five minutes to run because it is full of tests for trivial code or brittle implementation details is actively harmful: it slows exactly the feedback loop testing exists to accelerate, and teams under that pressure eventually stop running the suite locally, or start bypassing failures administratively, destroying the suite's value as a trusted signal regardless of how many tests it contains.

**Options:**

1. **Preserve every historical test indefinitely** — treat test removal as permitted only when the feature itself is deleted.
2. **Periodically evaluate and remove low-value tests** — treat each test's continued presence in the suite as something that must keep justifying itself.

**Trade-offs:**

[Strong Recommendation] **Periodic removal of tests with negative ROI.** Preserving every historical test guarantees no test is accidentally lost, but it dooms the suite to slow monotonically over the life of the project, since low-value tests never leave once added. Evaluating tests continually requires judgment and occasionally means removing a test that once had value but no longer earns its keep — a genuinely uncomfortable decision, but one with a real payoff: faster feedback, smaller maintenance surface, and a suite developers actually trust and run.

**When to choose:** Ask three questions of a candidate test: Does it catch realistic regressions? Does it survive reasonable refactoring? Would anyone notice if it disappeared? If the honest answer to all three is no, the test is a candidate for deletion regardless of how long it has existed in the repository.

**Common failure modes:**

Mock-heavy suites frequently reach a point where the bulk of each test file is mock configuration and only a few lines are the actual behavioral assertion. Such tests verify a specific interaction sequence rather than software correctness, and they are simultaneously expensive to maintain and weak at catching real defects — passing even when production behavior is broken, because the mocked collaborators were configured to return values the real dependencies never would.

**Example:** During an infrastructure migration, a team discovers three hundred integration tests verifying that database connection pools reconnect correctly after transient network drops. The tests rely on brittle local threading simulations and fail non-deterministically on a meaningful fraction of CI runs, forcing repeated pipeline restarts. Because connection retry logic has since been delegated entirely to a well-verified standard client library, an engineer deletes all three hundred tests. Build time drops significantly, flakiness disappears, and no production regression follows — the tests had been consuming enormous maintenance effort to verify behavior the application no longer implements itself.

---

### Why Smart Engineers Disagree

The debate over how much to test became especially visible in the 2014 "TDD is dead" discussion, publicly initiated by David Heinemeier Hansson. The core criticism was not that testing lacks value, but that certain widespread testing practices — pervasive mocking, exhaustive unit coverage regardless of risk, designs shaped primarily around testability rather than clarity — produce suites that are brittle and expensive without a corresponding gain in confidence. Proponents of strict test-driven development responded that these are failures of testing practice, not of testing itself: a poorly disciplined team will produce a poor suite under any methodology, and the answer is better testing judgment, not less testing.

Both positions describe real failure modes. Suites can absolutely become maintenance burdens that outweigh their value. Systems with too little testing absolutely accumulate undetected regressions. Neither extreme — test everything unconditionally, or treat testing as fundamentally suspect — survives contact with a large, long-lived codebase.

The Go community offers a concrete data point in this debate: many Go projects favor a modest number of straightforward integration tests over exhaustive unit tests for every exported function, reflecting a cultural preference for testing meaningful behavior at the layer where it is naturally verified rather than maximizing test count as a proxy for rigor. This is not an argument against unit testing broadly — it is evidence that a mature engineering culture can choose deliberately where to test, and choose not to test elsewhere, without an accompanying collapse in quality.

The position this handbook takes: do not measure a test suite by how many tests it contains. Measure it by the confidence it provides relative to what it costs to keep. Concentrate tests on meaningful behavior and genuine risk. Refactor code that is hard to test because its design is poor. Do not write tests that verify trivial delegation, someone else's framework, or private implementation details that have nothing to do with the software's actual contract. And recognize that deleting a test that no longer earns its place is as legitimate an engineering decision as writing a new one — a smaller suite that is trusted and run continuously outperforms a larger one that is slow and ignored.
