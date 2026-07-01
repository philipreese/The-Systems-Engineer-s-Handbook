# Ch 37 — Fixture-Based Testing

**Prerequisites:** [The Testing Pyramid](ch34-the-testing-pyramid.md), [What Belongs at Each Layer](ch35-what-belongs-at-each-layer.md), [When to Mock vs. Use Real Dependencies](ch36-when-to-mock-vs-use-real-dependencies.md), [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md)

**New vocabulary introduced:** fixture, state contamination, fixture bloat

**Key takeaways:**
- A fixture is the complete set of preconditions a test depends on: database records, in-memory objects, configuration, and environmental state. Shared mutable fixtures — datasets multiple tests draw from without isolation — are the primary source of order-dependent test failures.
- Static fixture files are appropriate only for genuinely immutable reference data. Factory functions and scoped fixture functions are the default for entity data, because they keep state creation local to the test that requires it.
- Fixture bloat — a shared dataset grown through incremental additions until no single engineer understands its full dependency graph — is structurally inevitable in large static fixture systems at scale. The remedy is not to build them for mutable data.
- State reset between tests is a separate decision from state construction. Transactional rollback is fast and sufficient for most integration tests. Truncation is required when tests involve asynchronous workers or multiple database connections that commit outside the test thread's transaction.
- The data configurations that directly determine whether a test passes or fails belong inline and visible. Infrastructure setup — connections, containers, authentication contexts — is legitimately shared through scoped injection. The two should not be conflated.

---

[Ch 36](ch36-when-to-mock-vs-use-real-dependencies.md) addressed whether dependencies should be real or replaced. This chapter addresses the state those dependencies contain: how to set it up, how to prevent it from coupling tests to each other, and how to reset it cleanly between runs.

Tests that pass individually but fail in a suite are almost always a fixture problem. The cause is *state contamination*: one test modified shared state and did not restore it, and the next test executed against a world that had been silently altered. The failure is non-deterministic — it depends on execution order — and invisible in the failing test itself, because the defect is in a different test that ran earlier.

---

### Decide How to Construct Fixture State

**What it is:** The choice of mechanism for supplying the known state a test depends on — whether that state arrives as static data files loaded before the suite, generated programmatically per test, composed through a fluent builder API, or injected through a framework's lifecycle management system.

**Why it exists:** Tests require coherent object graphs. A test that exercises invoice processing needs a customer record, a billing plan, and line items with valid relational associations. Constructing this graph from scratch inside every test is verbose; sharing it globally creates coupling. The fixture construction strategy is how teams navigate that tension.

**Options:**

1. **Static fixtures** — predefined datasets stored in files (YAML, JSON, SQL) loaded once before the suite runs and shared across all tests. The approach popularized by the original Rails fixture system.
2. **Factory functions** — programmatic helpers that instantiate and persist entities with sensible defaults, accepting overrides for the specific attributes the test cares about (`UserFactory.create(role="admin")`). Exemplified by Ruby's factory_bot and Python's factory_boy.
3. **Object builder pattern** — a fluent API for composing complex objects step by step (`NewOrderBuilder().withCustomer(...).withLineItem(price=100).build()`). Used where object construction is hierarchically complex and raw factory calls would obscure the relationship structure.
4. **Scoped fixture functions** — dependency-injected setup functions with explicit lifecycle scope (function, class, module, or session), each scope controlling when the fixture is constructed and torn down. The model pytest employs.

**Trade-offs:**

[Strong Recommendation] **Factory functions or scoped fixture functions as the default for entity data.** Both approaches keep state creation local to the test that requires it. A factory-based test declares exactly which entity attributes it depends on and constructs only what it needs. A scoped fixture function declares its dependencies by name, with the scope determining whether the fixture is reconstructed per-test or shared within a controlled boundary.

**Static fixtures** are appropriate when data is genuinely immutable: a fixed set of ISO country codes, a permission hierarchy that the application reads but never writes, a product category tree that every test uses identically. Used for mutable entity data, they become the source of fixture bloat.

**Object builders** add value when hierarchical entity relationships would make factory calls verbose or ambiguous. For simpler entities they add indirection without benefit.

**Scoped fixture functions** introduce a risk when scope is misapplied. A session-scoped fixture for a database connection or a container lifecycle is appropriate — that infrastructure is genuinely shared and immutable from the tests' perspective. A session-scoped fixture for mutable entity data recreates the shared-state problem with different syntax.

**When to choose:** Use factory functions for entity data whose attributes directly govern the outcome of a test assertion. Use scoped fixture functions for expensive infrastructure initialization — containers, database connections, authentication tokens — that is correctly shared across multiple tests. Use static fixtures only for reference data no test modifies.

**Common failure modes:**

*The Fixture Bloat Avalanche.* A shared YAML fixture file starts with three records. Over five years, fifty engineers add rows to cover specific edge cases — a premium user, a suspended account, an international billing address, a deleted organization. The file grows to two thousand lines of interdependent relational records. No engineer understands the full dependency graph. A developer renames an attribute in row 12 to fix a production bug; forty-five tests in unrelated modules fail immediately, because they implicitly depended on that exact string value through relational associations no one had mapped. The file cannot be safely modified — it can only be appended to.

*The Leaky Session Cache.* An engineer declares a search index fixture at session scope to avoid reinitializing it for every test. A test midway through the suite inserts records into the index and fails before its teardown code runs. The next test — in a completely different module — encounters leftover state its author never knew could exist. The failure is intermittent: it disappears when that test is run in isolation and reappears when the suite runs in a particular order.

**Example:** The Ruby community's migration from static YAML fixtures to factory_bot is the canonical demonstration of fixture bloat at scale. Large Rails applications accumulated fixture files that had become effectively unmaintainable: schema changes required updating hundreds of fixture rows, and tests coupled to shared state broke unpredictably during routine development. factory_bot solved the problem by moving data creation into code, co-locating each test's preconditions with the test itself. Python's pytest fixture system extends the model: instead of global data files, fixture functions carry explicit scope declarations that make shared lifetime visible in the test signature. Django addresses the same coupling through a structural default: the test database is reset between every test automatically, making accumulated shared mutable state physically impossible without deliberate override.

---

### Reset State Between Tests Explicitly

**What it is:** The decision about how to ensure that state written by one test does not persist into the execution context of the next — the isolation mechanism at the persistence layer.

**Why it exists:** Factory functions control how state is *created*; they do not control how state is *cleared*. An integration test that inserts records into a real database leaves those records behind when it completes. Without an explicit reset strategy, the second test starts with state from the first. The construction strategy and the reset strategy are separate decisions with separate trade-offs.

**Options:**

1. **Transactional rollback** — wrap each test in a database transaction and issue a `ROLLBACK` immediately after the test completes, reversing all writes without touching disk.
2. **Truncation or schema reset** — physically clear all non-reference tables (or drop and recreate the schema) between tests.

**Trade-offs:**

[Strong Recommendation] **Transactional rollback as the default.** Rolling back a transaction that inserted and modified hundreds of rows takes microseconds — no disk I/O, no DDL operations. Integration test suites built on transactional rollback can execute thousands of database-touching assertions without accumulating meaningful overhead. The mechanism works cleanly for any test whose operations stay within a single database connection and transaction context.

**Truncation is required** when the test exercises code that commits outside the test's transaction: background workers on separate threads, asynchronous tasks that use their own database connections, or code under test that explicitly issues commits. A transactional rollback cannot undo a write committed by a different connection. Django distinguishes these cases structurally with two test base classes: `TestCase` uses rollback and is fast; `TransactionTestCase` uses truncation, accepts the latency cost, and is used only when the scenario genuinely requires it.

**When to choose:** Default to transactional rollback. Switch to truncation when the scenario under test involves asynchronous workers, multiple database connections, or explicit transaction isolation that requires real commits.

**Common failure modes:**

*The Asynchronous Phantom Leak.* A test triggers an asynchronous task to update a user record. The main test thread completes its assertions, the transaction rolls back, and the test passes. Two seconds later — while the next test is actively running — the background worker finishes processing and writes its stale result to the database, corrupting the runtime context of an unrelated test. The failure is intermittent, disappears on rerun, and is nearly impossible to trace because the cause executed in a different thread during a different test.

**Example:** Django exposes the rollback-versus-truncation choice as a deliberate class hierarchy. `TestCase` wraps each test in a transaction and rolls it back on completion — fast by default. `TransactionTestCase` issues a full truncation between tests, required when the scenario involves Celery tasks, Sidekiq workers, raw SQL commits, or explicit multi-connection transaction isolation. The structural split makes the trade-off visible in the code rather than hiding it in framework configuration.

---

### Why Smart Engineers Disagree: Implicit Context vs. Explicit Inline Setup

The persistent disagreement in fixture design is between brevity and transparency.

Engineers who favor implicit context — scoped fixture injection, shared setup methods, module-level initialization — argue that forcing every test to construct its own state creates walls of boilerplate that obscure the behavior under test. If twenty tests require an authenticated admin user with an active subscription, constructing that entity inside every test body is repetition that harms readability.

Engineers who favor explicit inline setup argue that when fixture state is hidden behind injected parameters, a failing test cannot be understood without navigating to the configuration file that built the state. Debugging becomes archaeology. They accept setup code duplication as the cost of keeping each test self-contained — the failure should be diagnosable from the test body without opening another file.

```
# Implicit: data that may govern the assertion is hidden in an injected parameter
def test_billing(premium_org_user):
    assert premium_org_user.can_bill()   # What exactly is premium_org_user?

# Explicit: the data controlling the assertion is visible where the assertion lives
def test_billing():
    org = OrgFactory.create(plan="premium")
    user = UserFactory.create(org=org)
    assert user.can_bill()
```

The practical resolution is to separate these concerns by what they govern. Infrastructure setup — a database connection, a container, an authentication token — is legitimately shared and appropriately managed through scoped injection or a shared setup method; it does not change which branch of logic executes. Entity data — the specific user attributes, the specific order state, the specific pricing configuration — that directly determines whether an assertion passes belongs inline, in the test body, visible to whoever reads the failure.

Use implicit context for shared infrastructure. Use explicit construction for data that governs outcomes. The failure mode of each approach is the failure mode of applying it where the other belongs.
