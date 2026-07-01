# Ch 40 — Test Naming and Structure

**Prerequisites:** [The Testing Pyramid](ch34-the-testing-pyramid.md), [What Belongs at Each Layer](ch35-what-belongs-at-each-layer.md), [Fixture-Based Testing](ch37-fixture-based-testing.md), [When Not to Test](ch39-when-not-to-test.md), [Naming Conventions and When They Matter](../part04-code-organization/ch28-naming-conventions-and-when-they-matter.md)

**New vocabulary introduced:** behavioral specification naming, Arrange/Act/Assert, table-driven test

**Key takeaways:**
- A test name should describe the behavior under test and the condition that triggers it — "returns X when Y" — not the method being invoked. Behavior-oriented names survive refactoring; method-oriented names go stale the moment the method is renamed and reveal nothing when a failure appears in a CI log.
- Arrange/Act/Assert (or its behavior-driven equivalent, Given/When/Then) is the default internal structure for a test body: construct the preconditions, execute the operation, verify the outcome, each visually distinct. The vocabulary matters less than the consistency.
- Each test should verify one coherent behavior. A test combining several unrelated behaviors hides every failure after the first one and forces the reader to untangle multiple concerns to diagnose a single regression.
- Table-driven tests are the deliberate, named exception to one-behavior-per-test: many input/output examples of the *same* behavior belong in one structured table with one execution path, not one function per case.
- A test's name and structure should be optimized for the moment it is read under the least favorable conditions: a failing CI log, with no source file open. If the failure alone doesn't explain what broke, the test has not finished its job.

---

A test is written once and read many times, and most of those readings happen under pressure — a CI failure blocking a deploy, a production regression under investigation, an unfamiliar behavior surfacing mid-development. In that moment the test's name and structure are read before its body is, often instead of its body. A test that communicates its purpose in its name and its intent in its structure is documentation that survives the person who wrote it. A test that requires the reader to reconstruct intent from implementation before diagnosis can even begin has failed at the one job a test uniquely performs: turning a failure into an answer.

---

### Name Tests After Behavior, Not Implementation

**What it is:** The convention of naming a test after the observable behavior it verifies and the condition under which that behavior applies — `returns_null_when_user_id_does_not_exist`, `rejects_expired_session_token` — rather than after the method or class it exercises — `testGetUser`, `testSession`.

**Why it exists:** Implementation changes far more often than behavior. Methods get renamed, split, and reorganized; internal algorithms get replaced; the externally observable contract usually survives all of it unchanged. A test name bound to a method name goes stale the instant that method is renamed, even though nothing about correctness changed — and worse, when a test named after a method fails in a CI log, the failure communicates nothing beyond "something is wrong with this method." A name built from behavior and condition answers the two questions a reader actually has when a test fails: what was supposed to happen, and under what circumstances did it not.

**Options:**

1. **Behavior-based names** — describe the expected outcome and its triggering condition (`registers_new_user_when_payload_is_valid`, `calculates_sales_tax_for_out_of_state_orders`).
2. **Method-based names** — bind the test identifier to the production method under test (`testGetUser`, `test_RegisterUser_Valid`).
3. **Generic or numbered names** — arbitrary or sequential identifiers (`testOne`, `shouldWork`).

**Trade-offs:**

[Strong Recommendation] **Behavior-based names as the default**, for essentially all core domain logic, API endpoints, and service-layer code. They communicate intent immediately, survive refactoring that leaves behavior unchanged, and turn a failure message itself into a diagnosis — no source inspection required. The cost is real but small: behavior-based names run longer and require the author to state precisely what the test is checking, which is itself a useful forcing function. If a test cannot be named in terms of a behavior and a condition, that is frequently a sign the test's own purpose is not well defined.

**Method-based names** remain defensible in one narrow case: highly stable, low-level algorithmic primitives or pure mathematical utilities where the method name *is* an unvarying mathematical expression (`BitMap.set()`), and there is no meaningful behavior/condition distinction to state separately from the operation itself.

**Generic names** have no defensible use — they communicate nothing on failure and provide no documentation value at any point in the test's life.

**When to choose:** Default to behavior-based naming everywhere confidence about correctness matters. Reserve method-based naming for the narrow class of primitives where the method name already fully describes the behavior.

**Common failure modes:**

*The Mystery Failure Block.* A production regression triggers a failure in a test named `testUpdateInvoice3`. The engineer investigating the CI log sees only a stack trace and a name with no behavioral content. Understanding what actually broke requires reopening the test file, reconstructing the input payload and database state the original author had in mind, and inferring what invariant the assertions were meant to protect — turning a routine regression triage into an archaeology exercise.

**Example:** RSpec and Jest formalize behavior-based naming structurally through nested `describe`/`context`/`it` blocks rather than a single long identifier:

```javascript
describe("OrderProcessor", () => {
  context("when the item inventory is exhausted", () => {
    it("moves the line item to the backorder queue", () => {
      // test body
    });
  });
});
```

When this fails, the runner concatenates the nested descriptions into a single readable sentence — `OrderProcessor when the item inventory is exhausted moves the line item to the backorder queue` — giving the exact same diagnostic clarity as a long behavior-based function name, expressed instead through structural nesting.

---

### Structure Tests with Arrange, Act, Assert

**What it is:** The convention of organizing every test body into three visually distinct sections: constructing the required preconditions (Arrange), executing the operation under test (Act), and verifying the observable outcome (Assert).

**Why it exists:** A reader diagnosing a test asks the same three questions regardless of the test's subject: what was the starting state, what happened, and what was expected to result. AAA structure puts the code in the same order as those questions, so a reader can scan a test top to bottom and immediately identify which part is setup, which is the action under scrutiny, and which is the check that failed. Without an enforced structure, setup, execution, and verification blend together in whatever order they were written, and a reader must reconstruct which lines matter to the specific behavior being tested.

**Options:**

1. **Arrange/Act/Assert (AAA)** — a flat, three-part structure, typically separated by blank lines or comments, with no framework dependency.
2. **Given/When/Then (BDD)** — the same three-part structure expressed through nested framework constructs (`describe`/`context`/`it`, `beforeEach` blocks, Cucumber feature syntax).
3. **Unstructured** — setup, execution, and assertions interleaved in whatever order the test was originally written.

**Trade-offs:**

[Strong Recommendation] **AAA as the default structural baseline**, particularly for backend systems, strongly-typed ecosystems, and any code where a flat top-to-bottom read is the natural reading pattern. It requires no framework support, imposes no runtime cost, and reads identically regardless of language. Its weakness is that it depends entirely on team discipline — nothing enforces the separation beyond convention, and it can erode into interleaved code without active attention.

**Given/When/Then** structurally enforces the same separation through the test framework's own syntax, and it earns its keep specifically where behavior is multi-layered and cross-functional — front-end applications, customer journey flows, business rule engines with many interacting conditions — because the nesting can mirror the application's actual branching logic (authenticated vs. not, overdrawn vs. not, weekend vs. weekday). Its cost is real: deep nesting can obscure state (a reader must trace across multiple `beforeEach` blocks to know what exists at the assertion), and heavy closure allocation carries a runtime cost in some frameworks.

**Unstructured tests** have no advantage beyond avoiding trivial ceremony in genuinely tiny tests, and they degrade quickly as a codebase grows — setup, mutation, and assertion interleave freely, and a reader cannot tell preparation from verification without close reading.

**When to choose:** Default to flat AAA for backend systems, strongly-typed languages, and algorithmic or adapter code where behavioral variance is low relative to component stability. Choose Given/When/Then specifically when testing multi-layered, branching behavior — front-end flows, business rule engines, cross-functional customer journeys — where nested context blocks can map directly onto the application's actual decision tree. The choice of vocabulary matters far less than applying it consistently across the codebase.

**Common failure modes:**

*The Interleaved Assert Rollercoaster.* An integration test performs an action, asserts a row count, performs a second mutation, asserts an update, invokes a third service, and checks a final flag — all interleaved rather than separated. When the third assertion fails, the test stops immediately, concealing whether the fourth mutation was even reached and hiding the true state of the system from the engineer investigating the failure. The interleaving destroyed the ability to reason about what had and hadn't been verified.

**Example:**

```go
func TestInvoiceCalculation_withPremiumDiscount(t *testing.T) {
    // Arrange
    calculator := NewInvoiceCalculator()
    premiumUser := DomainUser{ID: 42, Tier: TierPremium}
    basePayload := OrderPayload{Total: 100.00}

    // Act
    result, err := calculator.Compute(premiumUser, basePayload)

    // Assert
    if err != nil {
        t.Fatalf("unexpected calculation failure: %v", err)
    }
    if result.FinalTotal != 80.00 {
        t.Errorf("expected 20%% premium discount total of 80.00, got %f", result.FinalTotal)
    }
}
```

The three sections are visually distinct through comments and blank lines alone — no framework machinery is required to make the structure legible.

---

### Test One Behavior at a Time

**What it is:** The principle that each test function should verify a single coherent behavior — accepting multiple assertions when they collectively describe that one behavior, but never combining assertions about genuinely unrelated behaviors in a single test.

**Why it exists:** A failing test should point at exactly one regression. When a single test asserts several unrelated behaviors — validation, then persistence, then logging, then notification — the first failing assertion halts execution and hides whatever the remaining assertions would have revealed. A reader investigating the failure cannot tell from the report alone whether the other behaviors are broken too; they must rerun, comment out assertions, or step through the debugger to find out. Splitting unrelated behaviors into separate tests means each failure report is already a complete, unambiguous diagnosis.

**Options:**

1. **One behavior per test** — a test function verifies a single coherent behavior, possibly with multiple assertions that all describe that behavior.
2. **Many unrelated behaviors per test** — a single test function asserts on several independent aspects of a broader operation.
3. **Table-driven or parameterized execution** — a single test function iterates over many input/output examples of the *same* behavior.

**Trade-offs:**

[Strong Recommendation] **One behavior per test.** The cost is more test functions and some duplicated setup across them; the benefit is that every failure is immediately diagnostic, tests can be reordered or run independently without hidden dependencies, and each test documents exactly one thing a reader can rely on. Large multi-purpose tests reduce duplicated setup but produce poor diagnostics and tangle multiple responsibilities into one pass/fail signal, which is a worse trade in nearly every case.

**When to choose:** A useful test is: if removing one assertion would leave the test verifying a completely different behavior, that assertion belongs in a separate test. If removing an assertion leaves the remaining assertions still coherently describing the same single behavior, they belong together.

**Common failure modes:**

A single test asserts that a request passes validation, is persisted, is logged, triggers authorization checks, and sends a notification. A failure in the first assertion — validation — prevents the remaining four from ever executing. Diagnosing the actual regression requires manually re-running the scenario while stripping out the earlier assertions to see how far execution actually gets, work the test suite exists specifically to avoid.

**Example:** A login test may reasonably assert both that authentication succeeds and that a session token is produced — these are two facets of one behavior, "a valid login establishes a session." The same test should not additionally verify password reset functionality; that is an unrelated behavior deserving its own test.

---

### Use Table-Driven Tests for Repeated Behavior Across Inputs

**What it is:** The pattern of expressing many input/output examples of the *same* behavior as entries in a data table, iterated by a single test function, rather than as separate hand-written test functions per example.

**Why it exists:** When dozens of examples verify identical behavior and differ only in their input data, writing one function per example produces overwhelming duplication — the vast majority of each function is repeated setup and invocation logic, with only the input and expected output actually varying. A table-driven test isolates the invariant execution mechanics from the varying data, so adding a new case means adding one row to a table rather than copying an entire function.

**Options:**

1. **Separate test function per example** — each input/output case gets its own independent, isolated test.
2. **Table-driven test** — a single function iterates over a structured collection of cases, each with its own input, expected output, and descriptive name, executing each as an isolated sub-test.

**Trade-offs:**

[Strong Recommendation] **Table-driven tests specifically when every case exercises the same behavior under different inputs** — this is the idiomatic Go pattern, and it applies equally well in any language to validators, parsers, and pure mapping functions. It is compact, trivially extended (one new struct entry rather than one new function), and encourages systematic enumeration of edge cases because adding one is nearly free. It is deliberately a *named exception* to one-behavior-per-test, not a violation of it: each table entry is still one behavior, expressed many times over.

**Separate functions per example** remain the right choice when different cases require substantially different setup or assertions rather than merely different data — forcing dissimilar cases into one table obscures rather than clarifies.

**When to choose:** Use a table when the only thing varying across cases is the data — same execution path, same assertion shape. Avoid tables when cases genuinely differ in what they set up or what they check; forcing that variation into table rows produces a table that no longer represents one coherent behavior.

**Common failure modes:**

*The Anonymous Loop Crash.* An engineer writes a custom loop over test cases without registering each case as an isolated sub-test (Go's `t.Run`, or an equivalent per-case registration in other frameworks). When one case panics — an out-of-bounds index, an unhandled nil — the entire test function aborts immediately, and the CI report shows only that the function failed, with no indication of which of the remaining untested cases would have passed or failed. The table's whole purpose — systematic, independent coverage of many cases — is defeated by the missing isolation.

**Example:** Go's standard library convention pairs a table of structs with `t.Run` to preserve independent execution and reporting per case:

```go
func TestUrlValidation(t *testing.T) {
    tests := []struct {
        name        string
        inputURL    string
        expectValid bool
    }{
        {"valid secure path", "https://google.com", true},
        {"missing scheme", "google.com", false},
        {"malformed protocol", "ftp://invalid-server", false},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            isValid := CustomUrlValidator(tc.inputURL)
            if isValid != tc.expectValid {
                t.Errorf("failed parameter: %s; expected validity %t, got %t", tc.inputURL, tc.expectValid, isValid)
            }
        })
    }
}
```

Each case runs and reports independently through `t.Run`, so a panic or failure in one case never masks the outcome of the others — the table's entire value depends on that isolation being present.

---

### Why Smart Engineers Disagree: Flat Files vs. Nested Context Trees

Most experienced engineers agree on the underlying principles this chapter argues for: name tests after behavior, structure them clearly, keep each one focused. The genuine disagreement is about *how* to express that structure at scale — specifically, whether test organization is better served by flat, self-contained functions or by deeply nested context hierarchies.

Engineers who favor flat structure — a cultural default in Go, Rust, and other systems-level communities — argue that nested `describe`/`context`/`it` trees force a reader to reconstruct an implicit execution context by scanning across multiple levels of `beforeEach` and setup blocks before they can know what state actually exists at a given assertion. They accept some duplicated setup across flat functions as the price of a test file that reads sequentially, top to bottom, as a self-contained document with nothing implicit.

Engineers who favor nested context trees — a cultural default in the JavaScript, Ruby, and broader BDD-influenced communities — argue that real-world behavior is often genuinely hierarchical: a system behaves one way when authenticated, differently again when an account is overdrawn, differently still on a weekend. Expressing that hierarchy as flat functions means replicating large blocks of near-identical setup across many independent tests, and when a shared precondition changes, every one of those duplicated blocks must be found and updated by hand.

The resolution turns on the actual shape of the behavior being tested, not a universal preference. Where behavioral variance is genuinely hierarchical and dense — many interacting conditions, deep branching, cross-functional user journeys — nested context trees map the test structure directly onto the application's real decision tree, and the alternative is unsustainable duplication. Where the underlying risk is concentrated in linear data transformations, adapters, or algorithmic calculations, nesting adds abstraction that obscures more than it clarifies, and a flat AAA structure — with table-driven tests absorbing genuine input variation — keeps the suite legible. Choose based on the density of behavioral branching relative to the stability of the component under test, not on which convention is more familiar.

---

**Position:** Name tests as behavioral specifications, not implementation checks. Structure them consistently with AAA or its BDD equivalent. Keep each test focused on one coherent behavior, using table-driven tests as the explicit, named exception when many examples share that behavior. The objective is not merely executable tests — it is failures that are understandable the moment they appear, before a single line of implementation is read.
