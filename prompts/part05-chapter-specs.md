# Part V — Chapter Specifications

Pre-filled special instructions for each Part V chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part V: Part I (especially Ch 02: complexity, Ch 04: abstraction and
information hiding, Ch 07: reliability) and Part IV (especially Ch 29: file organization,
Ch 32: error handling). Several chapters also build directly on Part II architecture content —
specifically, the hexagonal/ports-and-adapters architecture from Ch 11, which determines
where tests naturally live.

One forward reference from Part IV must be honored here:
- Part IV Ch 29 covered test file co-location briefly and explicitly deferred the full
  treatment — "this is a Part V (Testing Strategy) concern at its core" — to Ch 35.

---

## Ch 34 — The Testing Pyramid

```
This chapter establishes the structural model that anchors the rest of Part V. Every
subsequent chapter — what belongs at each layer, when to mock, how to name tests, what
coverage means — assumes the pyramid as the frame of reference. Build it carefully here so
it does not need to be re-explained later.

Cover the three layers explicitly: unit tests (many, fast, isolated, no I/O), integration
tests (fewer, slower, real infrastructure or close substitutes, crossing one or more
component boundaries), and end-to-end tests (fewest, slowest, full-stack, closest to a real
user interaction). Name why the pyramid is shaped the way it is: tests at the base are cheap
to write, fast to run, and give immediate feedback when they fail; tests at the top are
expensive to write, slow to run, and when they fail they tell you something broke somewhere
without telling you where.

Cover the ice cream cone anti-pattern directly — the inverted pyramid where a codebase has
many slow end-to-end tests, a thin layer of integration tests, and few or no unit tests. This
is the canonical failure mode of teams that adopt testing without a principled model. E2E
tests that cover behavior that could have been unit-tested are not a safety net — they are a
maintenance tax that makes the suite too slow to run in CI and too brittle to trust.

Be explicit that the pyramid is a model, not a formula. The right ratio of unit to integration
to E2E tests depends on the system's architecture — a backend API service and a UI-heavy
single-page application will have different natural pyramids. The model's value is the
directional principle: push as much test coverage as low in the pyramid as the architecture
permits.

Anchor in real systems: Mike Cohn's original testing pyramid (Succeeding with Agile, 2009)
as the source of the model; Google's internal categorization of tests as Small, Medium, and
Large (documented in "Software Engineering at Google") as a concrete institutional
implementation of the same principle; the documented failure mode of large E2E test suites
becoming so slow and flaky they are disabled or bypassed in CI.

Do NOT cover: what specific test types belong at which layer (Ch 35 — that is the next
chapter's entire job); mocking and test doubles (Ch 36); fixture setup (Ch 37); test naming
and structure (Ch 40); coverage metrics (Ch 41). This chapter establishes the model; those
chapters answer specific questions within it.
```

---

## Ch 35 — What Belongs at Each Layer

```
Given that Ch 34 established the pyramid structure, this chapter answers the practical
question: for any given piece of code, which layer of the pyramid should test it? This is
where the abstract model becomes actionable.

Cover what belongs at the unit layer: pure functions, data transformations, domain logic,
business rules, validation. These are the things that have no I/O, no external dependencies,
and whose correctness depends only on inputs and the logic that transforms them. They are
the cheapest to test and the most valuable to test thoroughly at this layer.

Cover what belongs at the integration layer: database queries and ORM behavior (test against
a real or near-real database, not a mock), cross-component behavior where multiple
collaborators interact, adapter implementations in the hexagonal architecture from Part II
Ch 11 (the port contracts are validated at this layer, not mocked away), and external service
contracts where integration behavior is the thing being verified.

Cover what belongs at the E2E layer: critical user journeys that span the full stack, the
handful of paths through the system where the specific sequence of component interactions is
itself the thing being validated. E2E tests should be few by design, not by neglect.

Cover what the layer boundary question reveals about system design: code that is difficult to
unit test — because it mixes business logic with I/O, because it has hidden dependencies,
because it requires extensive setup — is usually revealing a design problem in the code
itself. The hexagonal architecture from Part II Ch 11 is the structural answer: isolate
business logic in the core, push I/O and infrastructure to adapters, and the natural test
structure follows. Domain logic is unit-tested; adapters are integration-tested; the ports
(interfaces) are the seams where the two meet.

Honor the forward reference from Part IV Ch 29: cover test file co-location — a test file
living next to its source file versus a parallel test tree — as a deliberate organizational
choice, not an afterthought. Co-location is the natural expression of package-by-feature for
tests: the test that covers an order module lives next to the order module.

Anchor in real systems: Go's convention of placing `_test.go` files in the same directory
as the code they test (and Go's further convention of `_test` package suffix for black-box
tests) as a language-level endorsement of co-location; pytest's discovery model as an
example of the parallel test tree approach; the hexagonal architecture test pattern where
the domain core is tested with pure unit tests and adapters are tested with Testcontainers
or equivalent.

Do NOT cover: the pyramid structure itself (Ch 34); mocking decisions — which dependencies
to replace and with what (Ch 36); fixture data setup (Ch 37); property-based input
generation (Ch 38); test naming and structure (Ch 40); coverage measurement (Ch 41).
```

---

## Ch 36 — When to Mock vs. Use Real Dependencies

```
This is the most contested practical question in testing and the source of the oldest
unresolved school-of-thought split in the field. Cover it honestly as a genuine trade-off,
not as a settled question, but take a position at the end.

Cover the test double taxonomy precisely and early — these terms are widely misused and the
confusion causes real communication failures on teams. A stub returns canned values; a mock
asserts that specific calls were made; a fake is a real but lightweight implementation (an
in-memory repository, a local SMTP server); a spy records calls for later assertion. Martin
Fowler's "Mocks Aren't Stubs" (2004) is the canonical reference and should be cited
explicitly.

Cover the two schools of thought: the classicist (Detroit) school, which favors using real
objects wherever possible and only replaces dependencies that are genuinely external to the
system under test (databases, external APIs, the clock, the filesystem); and the mockist
(London) school, which replaces all collaborators with mocks to isolate the unit under test
completely, testing interactions rather than outcomes. The classicist position produces tests
that verify real behavior; the mockist position produces tests that verify that the code calls
the right things in the right order, which breaks whenever the implementation is refactored
even if the behavior is unchanged.

Take a position: prefer real or near-real dependencies wherever the cost is acceptable.
Replace a dependency with a test double when it is external to your control (third-party
APIs, external payment processors), when it introduces non-determinism you cannot control
(the real clock, random number generators, network flakiness), or when its real version is so
expensive to stand up that it would make the suite impractical (a mainframe, a licensed
third-party service). The in-memory fake is usually better than a mock for persistence: an
in-memory repository implements the same interface as the real one, tests the same behavior,
and runs fast without spinning up infrastructure.

Cover Testcontainers as a significant shift in where the cost threshold for "real
dependencies" now sits: spinning up a real PostgreSQL instance in CI via a Docker container
is now cheap enough that mocking the database is often worse than just using the real thing.

Anchor in real systems: Martin Fowler's "Mocks Aren't Stubs" (2004) for the taxonomy and
school-of-thought split; the in-memory repository pattern common in domain-driven design
implementations as the canonical example of a fake; Testcontainers (Java, Go, Python) as
the specific technology that shifted the cost of real dependencies in CI; the specific failure
mode of mock-heavy test suites that pass when the code is broken because the mocks return
values the real dependency never would.

Do NOT cover: what layer a test belongs at (Ch 35); fixture data setup separate from the
dependency question (Ch 37); property-based testing (Ch 38); coverage (Ch 41).
```

---

## Ch 37 — Fixture-Based Testing

```
This chapter is about the data and state that tests depend on — how to set it up, how to
keep it from becoming the dominant maintenance burden in the test suite, and what patterns
have emerged from experience with large test codebases.

Cover the core tension: tests need known state to be deterministic, but shared fixture state
— global data that all tests depend on — couples tests to each other in ways that make
failures hard to diagnose and order-dependent. A test that passes in isolation but fails in a
suite is almost always a fixture coupling problem.

Cover the four main approaches: static fixtures (YAML or JSON files loaded before the suite
runs, shared across all tests — the Rails fixture model), factory functions (per-test data
created programmatically with sensible defaults, overriding only what the test cares about),
the object builder pattern (fluent builder API for composing complex test objects), and
pytest-style fixture functions (dependency-injected setup with explicit scope — function,
class, module, or session — controlling when setup and teardown occur).

Take a position: factory functions or fixture functions with explicit scope are the default;
static shared fixtures are acceptable for reference data that genuinely does not change
(a fixed set of country codes, a static product catalog used identically across all tests)
but are the wrong tool for entity data that tests need to vary.

Cover the fixture bloat failure mode: a fixture file that started as a few rows and now
contains hundreds of interrelated records that no single test uses in full. Every test
implicitly depends on the entire fixture, yet no test can be understood without knowing the
full fixture state. Changes to the fixture break unrelated tests. This is the static fixture's
failure mode at scale.

Anchor in real systems: Rails fixtures (static YAML, the original approach) and factory_bot
(the Ruby community's evolution away from static fixtures toward factory functions) as the
canonical before/after showing the real-world failure of static fixtures at scale; pytest's
fixture system as the most ergonomic modern approach to scoped setup/teardown; Django's
test database reset-per-test behavior as a structural solution to fixture isolation.

Do NOT cover: which layer the fixture lives at (Ch 35); whether to mock the dependency that
the fixture replaces (Ch 36); property-based input generation, which is the alternative
paradigm for the same data-construction problem (Ch 38).
```

---

## Ch 38 — Property-Based Testing

```
Property-based testing (PBT) is a supplement to example-based testing, not a replacement.
This chapter's job is to explain what it is, why the failure cases it finds are genuinely
different from those that hand-crafted examples miss, and precisely when it is the right
tool.

Cover the core idea: instead of asserting that `f(3)` returns `7`, you assert that for all
integers `n`, `f(n)` satisfies some property — for example, that `encode(decode(x)) == x`
for all `x`, or that a sort function always returns a list of the same length as its input
with the same elements. The testing framework generates hundreds of random inputs, finds the
first one that violates the property, and then shrinks it to the minimal failing case.
Shrinking is what makes PBT practical: the framework doesn't just tell you something broke,
it finds you the simplest possible input that breaks it.

Cover when PBT finds bugs that example-based tests miss: pure functions with complex or
large input domains (parsers, serializers, compressors, encoders), round-trip invariants
(encode/decode, serialize/deserialize, compress/decompress), algebraic properties
(commutativity, associativity, idempotency), and data structure invariants (a balanced tree
remains balanced after insertion). These are domains where the space of possible inputs is
too large to cover manually and where the property is more naturally expressed as a
universal statement than as a set of examples.

Cover when PBT is not the right tool: complex stateful systems where the interaction
sequence matters as much as the inputs, UI behavior, or any domain where defining a precise
invariant is harder than writing the examples. PBT requires that you can state what "correct"
means for all inputs — which is often straightforward for algorithmic code and difficult for
business logic with many special cases.

Anchor in real systems: QuickCheck (Haskell, 1999, John Hughes and Koen Claessen — the
origin of the model and the shrinking technique); Hypothesis (Python) as the most widely
adopted modern implementation, notable for its persistent failure database and
example database; fast-check (JavaScript/TypeScript); the real-world case of Ericsson using
QuickCheck to find bugs in Erlang's protocol implementations that years of manual testing
had not found — a frequently cited demonstration of the technique's power on production
systems.

Do NOT cover: the fixture-based approach to generating test data (Ch 37 — PBT generates
inputs from a specification, not from pre-built fixtures; these are different problems);
example-based test structure and naming (Ch 40); coverage (Ch 41).
```

---

## Ch 39 — When Not to Test

```
This is the most honest chapter in Part V. Every other chapter argues for testing something.
This one argues that some code should not be tested, some tests should not be written, and
some existing tests should be deleted. The failure mode this chapter addresses is a test
suite that imposes more maintenance burden than the confidence it provides justifies.

Cover the specific categories of code where tests add little value: trivial delegation
(a one-line getter or setter where the test would be longer than the code); tests that
validate the framework or language rather than your logic (asserting that Spring wires a
bean correctly, that an ORM generates the expected SQL); tests that test private
implementation rather than public behavior and break on every refactor that preserves
behavior; and tests where the only realistic failure scenario is a bug in the language
runtime or standard library.

Cover the specific signal that code is worth not testing: if writing a test requires more
setup than the actual assertion, and the setup is testing infrastructure you did not write,
consider whether the test is validating your code or someone else's. If the test would have
to be rewritten every time the internal structure of the function changes — even when the
behavior the function exports to callers doesn't change — the test is coupled to
implementation rather than behavior.

Cover the distinction between "hard to test" and "not worth testing." Code that is hard to
test is usually revealing a design problem: mixed concerns, hidden dependencies, poor
boundaries. Hard-to-test code is worth making testable by improving the design. Code that is
easy to test but tests the wrong thing — implementation details, framework wiring, trivial
one-liners — is where the "don't test" judgment belongs.

Cover the cost dimension explicitly: tests have a maintenance cost. A test that breaks on
every refactor regardless of whether any observable behavior changed is not free. A suite
that takes 45 minutes to run because it is full of tests for trivial code is actively
harmful — it slows down the feedback loop that testing is supposed to accelerate. Deleting
tests that have negative ROI is a legitimate engineering decision.

Anchor in real systems: the "TDD is dead" debate (David Heinemeier Hansson, 2014) as the
public articulation of the costs this chapter addresses — not endorsing the conclusion but
treating the critique as a real and specific one worth engaging; the specific documented
failure mode of mocking framework abuse producing test suites where 90% of the test code
is mock setup and 10% is assertion; the Go community's cultural preference for integration
tests over unit tests for straightforward code as a concrete counterpoint to unit-test
absolutism.

Do NOT cover: coverage thresholds and how to set them (Ch 41 — this chapter is about the
decision to not write a test at all, not about what percentage of lines to cover); what
layer to put a test at (Ch 35); the specific mechanics of fixture setup (Ch 37).
```

---

## Ch 40 — Test Naming and Structure

```
Tests are read more often than they are written — most often when they fail. The name of a
failing test is the first signal about what broke. A test named `testGetUser` tells you a
test failed; a test named `returns_null_when_user_id_does_not_exist` tells you what broke
and whether the behavior is acceptable. This chapter is about making the read experience
match the write intent.

Cover test naming as behavior specification: test names should describe the behavior under
test and the condition under which it is being tested, not the method being called. The
pattern "does X when Y" or "returns Z given W" is more useful than "testMethodName" because
it survives refactoring — if the method is renamed, the behavior it tests may not change,
and neither should the test name. If a test's name cannot be stated in terms of behavior
and conditions, that is often a signal that the test does not have a clear purpose.

Cover the Arrange/Act/Assert (AAA) structure as the default internal layout for a test:
setup the state required (Arrange), execute the operation being tested (Act), verify the
outcome (Assert). The three sections should be visually distinct — either by blank lines or
explicit comments — so a reader can immediately identify what the test is doing and what
it is checking. Cover the Given/When/Then BDD variant as the same structure with different
vocabulary, common in Cucumber, RSpec, and Jest's describe/it convention.

Cover the one-behavior-per-test principle: a test that asserts multiple unrelated things
in a single test function is harder to diagnose when it fails because a single failure
stops the test, hiding the state of all subsequent assertions. Tests are cheapest when they
are small and focused. The exception — which should be named explicitly — is Go's
table-driven test pattern, where a single test function iterates over a slice of test cases,
each case being a specific input/output pair. This is the idiomatic Go counterpoint to the
one-function-per-case convention elsewhere.

Anchor in real systems: Bill Wake's "Arrange Act Assert" (2001) as the origin of the
canonical terminology; RSpec's describe/context/it hierarchy as the most explicit expression
of the behavior-specification naming convention; Go's table-driven tests (`t.Run` with a
slice of cases) as the most widely used exception to one-behavior-per-test; Jest's
describe/it blocks as JavaScript's mainstream expression of the same conventions.

Do NOT cover: what code belongs at which test layer (Ch 35 — this chapter is about the
internal structure of tests, not where they live); fixture data setup (Ch 37); coverage
metrics (Ch 41); identifier naming for production code (Part IV Ch 28 — test naming is
a distinct convention with distinct rules).
```

---

## Ch 41 — Coverage: What It Measures and What It Doesn't

```
Coverage is the most widely misunderstood metric in software testing. Teams treat high
coverage as a quality signal and enforce coverage thresholds in CI. This chapter's job is to
explain precisely what coverage measures, what it cannot measure, and what the evidence
says about its relationship to test suite effectiveness.

Cover the measurement hierarchy: line coverage (was this line executed by any test) is the
weakest form — a line being executed is not the same as its behavior being verified. Branch
coverage (was each branch of every conditional executed) is stronger — it catches the case
where both branches of an `if` run but only one is tested. Mutation testing (does changing
the code's behavior cause a test to fail) is the strongest available signal — it verifies
that tests are actually checking outcomes, not merely executing code. Each level is more
expensive than the last.

Cover what high coverage does not mean: a suite with 95% line coverage and no assertions —
only executions — would report 95% coverage and catch zero bugs. A test that calls a
function and discards the result contributes to coverage without verifying anything.
Coverage measures execution, not verification. A suite can have 100% line coverage with
tests that are completely worthless.

Cover what low coverage does mean: a file with 20% coverage has large regions of code that
no test exercises under any condition. That is a meaningful signal — the behavior in those
regions is completely unverified. Coverage is a useful floor indicator (low coverage is
clearly bad) but not a useful ceiling indicator (high coverage does not mean the tests are
good).

Cover the coverage gaming failure mode directly: when teams enforce a numerical threshold
in CI, engineers under schedule pressure will write tests whose sole purpose is to execute
lines, not verify behavior. These tests add maintenance burden without adding confidence.
The threshold has been met; the behavior is no more verified than before. The mandate
produced compliance, not quality.

Cover mutation testing as the correct tool when you want to know whether your tests are
actually checking behavior — at the cost of a test run that is orders of magnitude slower,
since it requires re-running the suite for each mutation. Use it for auditing the most
critical modules, not as a gate on the entire codebase.

Anchor in real systems: Google's published research finding that line coverage is not
strongly correlated with test suite defect-detection effectiveness ("An Empirical Analysis
of the Python Package Index (PyPI)", and related testing research published by Ciera
Jaspan and colleagues); PIT (mutation testing for Java) and mutmut (Python) as the tools
that implement mutation testing in practice; the specific documented outcome of coverage
thresholds producing test suites full of assertion-free tests.

Do NOT cover: the testing pyramid and what tests to write (Ch 34, Ch 35 — coverage is a
measurement tool, not a design tool); property-based testing (Ch 38 — a different approach
to the same coverage gap problem); when not to write tests (Ch 39 — coverage thresholds
are one driver of unnecessary tests, but Ch 39 covers the full decision).
```
