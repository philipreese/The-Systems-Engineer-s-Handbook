# Ch 31 — When Abstractions Help vs. When They Obscure

**Prerequisites:** [Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md) (specifically: the wrong abstraction and the Rule of Three), [Dependency Direction and Inversion](../part02-software-architecture/ch12-dependency-direction-inversion.md), [Abstraction Layers: When to Add One](../part02-software-architecture/ch14-abstraction-layers-when-to-add-one.md)

**New vocabulary introduced:** speculative generality, indirection tax

**Key takeaways:**
- Every abstraction carries an indirection tax: a reader must now jump between an invocation and its execution, consuming working memory at every hop.
- Abstractions for readability and abstractions for flexibility solve different problems and must be evaluated by different criteria. A single-implementation wrapper that gives a clumsy API a domain-meaningful name is justified even if the underlying implementation never changes.
- An interface with exactly one implementation and no credible second one hides nothing. It doubles the type count and breaks IDE navigation for no architectural benefit.
- The Strategy Pattern applied to a choice that is fixed at compile time adds the overhead of runtime polymorphism while eliminating the readability of a plain switch statement.
- Rob Pike's Go proverb: "A little copying is better than a little dependency." Duplicating five lines of stable concrete code is often cheaper than extracting a shared abstraction that couples two call sites which have independent reasons to change.

---

[Ch 04](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md) established that the wrong abstraction is worse than no abstraction. [Ch 14](../part02-software-architecture/ch14-abstraction-layers-when-to-add-one.md) applied that test at the architectural layer level. This chapter applies the same test at the smallest grain: a single function, a single class, a localized utility. The failure modes here are distinct from architectural ones — not a misplaced layer, but an interface introduced for a variation that will never happen, a pattern applied where a plain conditional would have been clearer.

---

### Abstract for Readability vs. Abstract for Flexibility

**What it is:** The two legitimate motivations for introducing an abstraction — and the reason they must be evaluated differently.

**Why it exists:** Judging a readability abstraction by whether it supports multiple implementations is the wrong question. Judging a flexibility abstraction by whether it clarifies local reading is equally wrong. Conflating the two causes engineers to reject wrappers that improve code clarity and to introduce interfaces that serve no real architectural purpose.

**The two categories:**

**Flexibility abstractions** hide an implementation choice behind a stable contract so the implementation can be swapped — for testing, for multiple runtime behaviors, for future substitution at a real variation point. These earn their indirection by isolating genuine change. Evaluated by: does realistic variation exist, or is this speculation?

**Readability abstractions** translate a noisy, awkward, or implementation-heavy call into the domain's own vocabulary. A one-line wrapper that gives a cluttered third-party API call a meaningful name belongs here. Evaluated by: does this make the calling code read as a requirements statement rather than a library manual?

[Strong Recommendation] A readability abstraction is justified even with exactly one implementation, because its payoff is local clarity, not future substitutability — and it should never be held to a swappability bar it was never trying to clear.

**Example (readability):** A third-party SDK exposes `ThirdPartySdk.ExecuteWithTransaction(context, opts, callback)`. Wrapping it in `settleInvoice(invoiceId)` expresses domain intent at every call site and shields billing logic from the SDK's calling convention. The wrapper is correct even if `settleInvoice` will always delegate to exactly that one SDK forever.

**Example (readability, concrete):** A DynamoDB query via `boto3` requires assembling 15 lines of `ExpressionAttributeValues` and `KeyConditionExpression` dictionaries. Dropping this into the middle of `approve_loan()` destroys reading flow. Wrapping it in `find_customer_by_id(id)` is the right use of abstraction — not because the database will change, but because the loan approval logic should read like loan approval logic.

---

### Introduce an Interface Only for Credible Variation

**What it is:** The specific code-level smell of an interface whose sole implementation has existed unchanged since the interface was created, with no credible second implementation on the horizon.

**Why it exists:** The [Dependency Inversion Principle](../part02-software-architecture/ch12-dependency-direction-inversion.md) is correctly applied when a real variation point exists — a storage backend, a notification channel, a payment gateway — where the code genuinely benefits from separating what is done from how it is done. Misapplied at the function and class level, it produces interfaces that communicate no abstraction, hide no decision, and break IDE "jump to definition" navigation by adding a layer of indirection that leads only to a single class.

**Options:**

1. **Concrete type** — depend directly on the implementation.
2. **Interface with multiple credible implementations** — abstract behavior behind a stable contract where real variation exists.
3. **Speculative interface** — introduce an interface before any second implementation is realistic.

**Trade-offs:**

[Strong Recommendation] **Concrete types** are correct by default for internal domain logic, data transformations, and utility functions that don't cross an I/O boundary. Refactoring a concrete type to an interface later, when a second implementation actually appears, costs one refactoring. Paying the indirection cost speculatively, forever, for a variation that never arrives, costs more.

Interfaces with **multiple credible implementations** are correct at genuine variation points: a `Repository` interface with a real PostgreSQL implementation and a test in-memory implementation, an `EmailSender` interface used in production and replaced by a `FakeEmailSender` in tests. The variation is real; the interface earns its existence.

**Speculative generality** is the failure mode: introducing the interface not because variation exists but because it might, someday, possibly be useful. The indirection tax is paid immediately; the benefit may never arrive.

**Common failure modes:** *The Impl Suffix Anti-Pattern.* A codebase where every interface is paired with exactly one class suffixed `Impl`: `BillingManager` / `BillingManagerImpl`, `UserService` / `UserServiceImpl`, `InvoiceProcessor` / `InvoiceProcessorImpl`. The interface communicates no abstraction — it is a carbon copy of the class's public methods. Navigating to the definition of any method requires an extra hop through an interface that adds no information. This is the ritual of abstraction without the substance.

**Example:** FizzBuzzEnterpriseEdition is a real, widely cited satirical repository that solves the trivial FizzBuzz algorithm by introducing `StringReturner`, `FizzStrategy`, `BuzzStrategyFactory`, and dozens more interfaces and factories. Every abstraction exists without hiding any meaningful complexity. The humor works because the indirection is real and navigable — each hop is a genuine file, a genuine class — while serving no purpose. It is the clearest possible demonstration that indirection without variation is strictly a cost.

---

### Don't Apply Dynamic Patterns to Static Choices

**What it is:** The failure of using runtime polymorphism — the Strategy Pattern, dependency-injected interfaces — to model a behavioral choice that is actually fixed at compile time or startup configuration.

**Why it exists:** The Strategy Pattern is correct when the behavior must change dynamically during a running process — different payment processors selected per request, different serializers selected per content type. Applied to a choice that is resolved once at startup and never varies, it hides the actual execution path behind indirection that provides no flexibility at runtime, only overhead at read time.

**Options:**

1. **Dynamic strategy injection** — define an interface, inject a concrete implementation at runtime.
2. **Static branching** — use a `switch` or `if/else` that directly invokes the concrete implementation based on a flag.

**Trade-offs:**

[Legitimate Trade-off] **Dynamic strategy injection** is correct when the algorithm must change during a single running process based on user input, request content, or plugin registration. It fully decouples the calling code from the branching logic; adding a new strategy requires no change to the caller.

**Static branching** is correct when the choice is fixed for the lifetime of the process — bound to an environment variable, set at startup, or compiled in. It is linear, readable, and shows all possible execution paths in one place. The cost — violating the Open-Closed Principle on extension — is real but often acceptable when the set of strategies is small and stable.

**Common failure modes:** *The Hardcoded Strategy.* An engineer implements the Strategy Pattern for a `TaxCalculator` because the company might expand to other countries. The dependency injection framework is hardcoded to inject `USATaxCalculator` on startup. Three years later, the system still operates in one country. Every engineer who reads the tax calculation code must navigate through an interface and a DI configuration to find a concrete class that has never been swapped. The polymorphism is an illusion; the overhead is permanent.

**Example:** In many Spring Boot Java applications, `@Qualifier` annotations inject specific implementations of a strategy interface where that implementation is statically bound in the configuration and never varies per request. A direct invocation of the concrete class would have been significantly more readable and equally correct. The pattern added indirection; it added no flexibility.

---

### A Little Copying Is Better Than a Little Dependency

**What it is:** The explicit preference, in cases of uncertainty, for duplicating small concrete implementations over extracting a shared abstraction that couples independent call sites.

**Why it exists:** Generalization requires predicting which future changes will be shared across callers. Those predictions are frequently wrong. Two call sites that look similar today often have independent reasons to diverge tomorrow. Extracting a shared abstraction couples them: when one caller needs a change, the shared abstraction must accommodate both, and the solution is usually another parameter.

**Options:**

1. **Duplicate the concrete implementation** — allow similar code to evolve independently.
2. **Extract a shared abstraction** — centralize the behavior when the pattern recurs.
3. **Build a configurable framework** — generalize through parameters, callbacks, and extension points.

**Trade-offs:**

[Strong Recommendation] **Duplicate** small, stable code when the callers are independent and future evolution is uncertain. The [Rule of Three](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md) is the right threshold: duplication in two places is data; duplication in three genuinely similar cases is a pattern worth naming.

**Shared abstractions** are correct after repeated evidence shows that the duplicated logic genuinely changes together — that modifying it in one place would have meant modifying it everywhere. The abstraction should reflect actual usage, not anticipated usage.

**Configurable frameworks** — functions with many optional parameters, callbacks, and flags serving a single caller — are almost always the wrong answer. They pay the complexity cost of generalization while serving none of the benefit: a heavily parameterized function with one caller is not a library; it is a prediction that the second caller will arrive.

**Common failure modes:** A strict DRY purist identifies three similar lines across two call sites. They extract a shared function. Because the two call sites have slightly different needs, the extracted function acquires a boolean flag: `processData(includeHeader=true)`. One caller later needs a different delimiter; the flag expands to an enum. A second caller needs a different character encoding; a third parameter appears. The abstraction now satisfies neither caller cleanly, couples both, and is more complex than the duplicated code it replaced.

**Example:** Rob Pike's Go proverb — *"A little copying is better than a little dependency"* — is a recognized, authoritative industry position, not a permission slip for sloppy code. It specifically means: a five-line concrete block duplicated in two independent places is often cheaper to maintain long-term than a shared abstraction that couples those two places. When one caller's requirements change, the duplicated code forks naturally. The shared abstraction becomes a negotiation.

---

### Why Smart Engineers Disagree: DRY Absolutism vs. The Cost of Indirection

The persistent argument about micro-abstractions is between strict DRY adherents and engineers who treat indirection as a cost to be justified rather than a virtue to be pursued.

A strict DRY purist sees duplication and sees a defect. Any two blocks of code that look similar should be extracted, regardless of whether the callers have independent reasons to change or whether the abstraction can be named precisely. The line count goes down; the purist considers this progress.

An engineer focused on indirection cost sees the extraction differently: two call sites that happen to look similar today have been coupled through a shared abstraction they didn't need. The coupling isn't free. Every future change to either caller must now negotiate with the other through the abstraction's interface. Adding a parameter to handle the difference is the usual outcome.

Both positions agree that duplication is a maintenance burden. They disagree on the threshold of evidence required before paying for a shared abstraction. The correct threshold is the Rule of Three applied honestly: wait until three genuinely similar use cases appear before extracting. Two similar call sites is coincidence. Three is a pattern.

The language culture matters too. Enterprise Java ecosystems built around frameworks like Spring have historically tolerated high abstraction density — DI containers, interface hierarchies, factory classes — because the frameworks reward it. Go's culture deliberately pushes in the other direction: concrete types by default, interfaces introduced late, duplication tolerated. Neither is objectively correct; they reflect different calibrations of the same underlying trade-off.
