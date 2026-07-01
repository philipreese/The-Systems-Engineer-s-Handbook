# Ch 38 — Property-Based Testing

**Prerequisites:** [The Testing Pyramid](ch34-the-testing-pyramid.md), [What Belongs at Each Layer](ch35-what-belongs-at-each-layer.md), [Fixture-Based Testing](ch37-fixture-based-testing.md), [Complexity Is the Enemy](../part01-systems-thinking/ch02-complexity-is-the-enemy.md)

**New vocabulary introduced:** property-based testing, behavioral invariant, shrinking, round-trip invariance

**Key takeaways:**
- Property-based testing specifies a universal statement that must hold for all valid inputs and lets the framework find counterexamples automatically. It supplements example-based testing; it does not replace it.
- The inputs humans choose for example-based tests reflect what engineers expect to work. PBT removes that confirmation bias by generating inputs the engineer would never have written.
- Shrinking — the framework's automatic reduction of a failing input to the smallest case that still fails — is what makes PBT practical. A failure on a 50,000-character document is undebuggable; a failure on two characters is not.
- PBT delivers strong returns when correctness can be expressed as a universal structural property: round-trip invariants (`encode/decode`), algebraic laws (commutativity, idempotency), and data structure invariants (sorted output contains all input elements). It delivers poor returns on business logic with human-defined exceptions, UI behavior, or any domain where defining the invariant is harder than writing the examples.
- Partial properties still find bugs that example-based tests miss. A property does not need to be mathematically complete to justify its place in the suite.

---

Example-based tests verify specific scenarios: `sort([3,1,2])` returns `[1,2,3]`; `encode("abc")` produces `"YWJj"`. Each test covers one input. An engineer writing examples implicitly selects inputs they expect to work — edge cases they have thought of, representative values from the domain. The inputs they have not thought of are precisely the ones the code may fail on.

Property-based testing inverts this. Instead of specifying particular inputs and expected outputs, it specifies a *behavioral invariant* — a statement that must hold for all valid inputs — and delegates input selection to the framework. The framework generates hundreds or thousands of inputs automatically, looking for any that violate the invariant. When it finds one, it does not report a massive random payload; it reduces the failure to the smallest input that still breaks the property.

This technique was formalized by John Hughes and Koen Claessen in QuickCheck (Haskell, 1999), which established the core model: random input generation coupled with automatic shrinking. Its most influential production demonstration came from Ericsson, where QuickCheck found defects in Erlang protocol implementations that years of manual example-based testing had not. The framework generated sequences of network events — packet reordering, disconnections, buffer overruns — that no human would have assembled by hand, and found race conditions and lockups that only appeared under unusual-but-real combinations of conditions.

---

### Apply PBT to Code Where Invariants Can Be Stated

**What it is:** The decision to test a component by specifying properties that must hold for all valid inputs, rather than or in addition to specifying expected outputs for particular inputs.

**Why it exists:** Engineers have confirmation bias when selecting test inputs. They choose values that match their mental model of how the code works — the ones they expect to produce correct results. This works well for documenting known scenarios. It fails to discover what happens on inputs the engineer never considered: null bytes embedded in strings, empty collections, Unicode sequences that confuse string length calculations, floating-point boundary values. PBT delegates input generation to the framework, which has no model of what the code "should" do and is indifferent to engineering expectations.

**Options:**

1. **Example-based tests only** — enumerate specific inputs and verify specific expected outputs.
2. **Properties only** — define invariants over generated inputs; forgo documented examples.
3. **Both** — use example-based tests to document important scenarios and communicate expected behavior; use properties to explore the input space those examples leave uncovered.

**Trade-offs:**

[Strong Recommendation] **Both together.** Example-based tests serve a documentation function that properties do not: they record the specific cases engineers care about, communicate intent to future readers, and anchor verification in concrete expected behavior. Properties serve a coverage function that examples cannot: they probe regions of the input space the engineer never would have explored.

Three categories of code where properties deliver high returns:

- **Round-trip invariants** — any transformation with an inverse: `decode(encode(x)) == x`, `deserialize(serialize(x)) == x`, `decompress(compress(x)) == x`. Every serializer, codec, encryption wrapper, and protocol adapter has this structure. A single round-trip property exercises more of the implementation than dozens of hand-assembled example payloads.
- **Algebraic laws** — structural mathematical properties: commutativity (`f(a, b) == f(b, a)`), associativity, idempotency (`f(f(x)) == f(x)`). Sorting functions, caching layers, normalization routines, and set operations typically satisfy algebraic laws that can be expressed directly as properties.
- **Data structure invariants** — constraints that should hold regardless of input: a sort output contains exactly the same elements as its input; a balanced tree remains balanced after any sequence of insertions; a heap satisfies heap ordering after any modification.

**Property-based testing provides poor returns** when no meaningful invariant can be stated independently of specific examples:

- Business logic with arbitrary human-defined exceptions: payroll tax calculations with jurisdiction-specific exemptions, promotional discount rules, regulatory compliance edge cases.
- UI behavior, where correctness depends on visual layout and human interpretation.
- Complex stateful workflows, where what is "correct" depends on the specific sequence of prior events as much as on the current inputs.

The test for whether PBT belongs is practical: can you state what correctness means without referencing a specific input? If yes, a property can express it. If the only way to describe correctness is to enumerate cases, example-based tests are the right tool.

**When to choose:** Default to both: examples for documentation, properties for input space coverage. Apply properties specifically to parsers, serializers, compressors, encoders, mathematical utilities, and data structure implementations — the algorithmic domains where the input space is too large to enumerate and correctness is structurally defined.

**Common failure modes:**

*The Vacuous Assert Mirage.* An engineer applies PBT to a markdown parser but writes the invariant as `assert result is not None`. The framework generates ten thousand random documents. All pass — the parser never throws an unhandled exception, which is all the property measured. The test provides false confidence: it does not verify that the parsed structure matches the semantic content of any input. The invariant was too weak to detect any parsing incorrectness. The tests are green; the parser may be completely broken.

*The Tautological Spec.* An engineer writes a property for a discount calculation engine. Unable to state the invariant without duplicating the business logic, they copy the conditional branching structure from the production code into the assertion block. The framework generates hundreds of inputs and passes on all of them. When a bug is later introduced into the production logic, the copied assertion reproduces the same bug — the test confirms the wrong answer is consistent. The property is circular; it verifies nothing.

**Example:** A sorting function satisfies four structural invariants regardless of input: the output is ordered (each element is ≤ the next), the output length equals the input length, the output contains exactly the same elements as the input as a multiset, and sorting an already-sorted input changes nothing. These four properties collectively verify more behavior than a hundred handwritten arrays and catch entire categories of bug — dropped elements, corrupted duplicates, off-by-one boundary errors — that example tests miss. Hypothesis (Python) and fast-check (JavaScript/TypeScript) implement the same model for modern stacks; both include generators for standard types and automatically shrink failures.

---

### Use Frameworks That Shrink Counterexamples

**What it is:** The framework capability that converts a randomly discovered failing input into the smallest input that still fails — making randomly generated defects practically diagnosable.

**Why it exists:** A framework that finds a failure has found a failing input, not necessarily a useful one. If the framework generates a 50,000-element list and discovers that the sort function produces incorrect output, the developer receives a massive payload of noise with no indication of which element or combination triggered the failure. Shrinking addresses this by repeatedly attempting to remove or simplify parts of the failing input, retaining any simplification that still causes the failure. The result is the smallest case that demonstrates the defect.

**Options:**

1. **Frameworks with shrinking** — after finding a failure, the framework reduces it to a minimal counterexample before reporting.
2. **Frameworks without shrinking** — the raw generated failure is reported without reduction.

**Trade-offs:**

[Strong Recommendation] **Always use frameworks with shrinking.** Shrinking is not a convenience — it is the mechanism that makes PBT practical at the unit test layer. Without it, failures in large input domains produce debugging payloads that require manual minimization before the actual defect becomes visible, often consuming more time than fixing the bug itself. With shrinking, the minimal counterexample frequently reveals the defect immediately.

Hypothesis extends shrinking with a persistent failure database: when a counterexample is found and minimized, Hypothesis saves the minimal failing input to a local, version-controlled cache. On subsequent test runs, it injects previously discovered failures first, before generating new random inputs. A failure found in CI is not lost when the process exits — it survives as a permanent regression case, verified on every future run.

**When to choose:** Prefer established PBT frameworks over custom random generators specifically because their shrinking implementations have been refined over years. QuickCheck pioneered the model. Hypothesis, fast-check, and GoCheck implement variants appropriate to their respective ecosystems.

**Common failure modes:**

The most common consequence of using a framework without effective shrinking is premature abandonment. An engineer encounters a failure on a 5,000-character random input, cannot identify which part of the payload caused it within a reasonable debugging budget, marks the test as flaky, and disables it. The test was not flaky — it had found a real defect. The absent shrinking step made the failure unactionable within normal working constraints.

**Example:** A parser encounters a failure on a randomly generated document of 8,000 tokens. The shrinking algorithm repeatedly removes tokens and retries, halving the input repeatedly until the property still fails on `"("`. The minimal counterexample immediately suggests an unclosed-parenthesis handling bug. The original 8,000-token failure pointed at nothing; the two-character minimal case points at the exact issue. Hypothesis documents this reduction in its output, showing the discovered failure and the minimal reproducer side by side.

---

### Why Smart Engineers Disagree: Completeness vs. Pragmatic Coverage

The disagreement in property-based testing is about whether partial properties have value.

The formal specification view holds that a weak invariant is worse than no invariant at all. A property that verifies only that a sorted output has the same length as its input — but not that it contains the same elements — allows a function that replaces every element with zero to pass. If the invariant is not complete, it provides false confidence. This view demands comprehensive properties or none.

The pragmatic view holds that partial properties catch bugs that no example-based test would find, regardless of whether they catch all bugs. An invariant that asserts a parser never crashes on arbitrary Unicode input is a useful bug-finding tool even if it does not verify the semantic correctness of the output. Several overlapping partial properties — round-trip correctness, crash-freedom, length preservation — together provide coverage that is both broad and cheap to write.

Both observations are correct. A vacuous property that always passes is not useful. A partial property that catches a category of defect example tests miss is genuinely useful. The practical resolution is not to demand formal completeness but to demand that every property be strong enough to fail on at least some class of defect — and to combine multiple partial properties when no single one is sufficient.

The productive question is not "can I write a complete specification?" but "can I state something that must be true for all valid inputs and that would actually fail if the code were wrong?" When the answer is yes, that property belongs in the test suite alongside the examples. When the answer is no — when writing the property requires duplicating the production logic or when no structural invariant exists — example-based tests are the better investment.
