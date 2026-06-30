# Ch 28 — Naming Conventions and When They Matter

**Prerequisites:** [Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Coupling and Cohesion at the Architecture Level](../part02-software-architecture/ch13-coupling-cohesion-architecture-level.md) (specifically: ubiquitous language and bounded context vocabulary), [File and Module Structure](ch27-file-and-module-structure.md)

**New vocabulary introduced:** ubiquitous language, Hungarian notation, semantic predicate naming

**Key takeaways:**
- Naming is the cheapest form of information hiding: a good name communicates intent without any additional indirection, layers, or runtime cost.
- Names should come from the problem domain before the implementation — matching the ubiquitous language the business already uses.
- Hungarian notation was useful in untyped C environments; in any modern statically or gradually typed language it duplicates what the compiler already knows and lies when types change.
- Predicate prefixes (`is`, `has`, `can`) are one of the few naming conventions that carry real disambiguating value — they distinguish a boolean from an object without encoding memory layout.
- `camelCase` vs. `snake_case` is a style choice, not a correctness choice. It matters only for consistency, and consistency should be enforced by a linter, not debated.

---

Naming is the smallest unit of design. Before understanding an implementation, a reader encounters the identifiers — and a good name communicates enough that the implementation becomes confirmation rather than discovery. This chapter distinguishes the naming decisions that carry architectural weight from the conventions that are purely stylistic. The two are not equally important.

Comments as an alternative communication mechanism are covered in [Ch 30](ch30-comments-what-to-comment-what-not-to.md). Abstraction design — when a name attached to a new function is worth the added layer — is covered in [Ch 31](ch31-when-abstractions-help-vs-when-they-obscure.md).

---

### Name by Intent, Not Implementation

**What it is:** An identifier should describe what a thing *means*, not how it happens to be built.

**Why it exists:** Code is read far more often than it is written. A name that mirrors the implementation forces every reader to mentally reverse-engineer the business intent. A name drawn from the domain matches the vocabulary the business already uses — the same ubiquitous language that defines a [bounded context](../part02-software-architecture/ch13-coupling-cohesion-architecture-level.md). When code and domain share vocabulary, engineers and domain experts can talk about the same thing without translation.

**Options:**

1. **Domain-oriented names** — identifiers drawn from the problem space: `Order`, `Invoice`, `RetryPolicy`, `findEligibleCustomers()`.
2. **Implementation-oriented names** — identifiers describing mechanisms: `DataProcessor`, `Manager`, `Handler`, `iterateCustomerRecords()`, `fetchFromPostgres()`.
3. **Stale names** — identifiers that once described the implementation accurately and now don't: an `XmlParser` that now parses JSON, a `loadCache()` method that makes a remote network call.

**Trade-offs:**

[Consensus] **Domain-oriented names** are more resilient to change because the business concept outlives any particular implementation choice. Switching from a list to a hash set, from PostgreSQL to Cassandra, from an HTTP call to a message queue — none of these change what `activeUsers` means to a reader. The only cost is that the name requires understanding the domain, which is a cost the engineer was going to pay anyway.

**Implementation-oriented names** are easy to invent and initially accurate, but they couple the reader's mental model to the current mechanism. When the mechanism changes, the name becomes a liability. Classes named `Manager`, `Handler`, `Processor`, or `Engine` impose almost no semantic constraints and reliably accumulate unrelated responsibilities because the name never pushes back.

**Stale names** are the degenerate case: the implementation changed, the name didn't, and now every reader is deceived.

**When to choose each:** Use domain vocabulary for anything representing a business concept. Reserve implementation-oriented names for the narrow infrastructure layer where the technical role genuinely is the domain — a `postgresDriver` variable inside a database connection pool, where the implementation itself is the point.

**Common failure modes:** An engineer names a variable `customerListArray`. Six months later, the type changes to a `HashSet` to enforce uniqueness. Because the data structure was encoded in the name, every reference must be renamed — a localized type change becomes a sprawling, risky pull request. This is the refactoring ripple: the name coupled every call site to the implementation detail it was supposed to hide.

**Example:** The Extreme Programming and Clean Code traditions center on intention-revealing names. `findEligibleCustomers()` hides the search mechanism; `iterateCustomerRecords()` leaks that a linear scan is occurring. When the scan becomes an index lookup, the second name is wrong. The first name survives the change unchanged.

---

### Don't Encode Type Information in Names

**What it is:** The decision of whether to prefix or suffix an identifier with metadata about its type or memory layout.

**Why it exists:** Hungarian notation originated in the C-era Windows API (`lpszName` for "long pointer to a null-terminated string") when compilers provided no type visibility and editors had no autocomplete. Encoding the type directly into the name was the only way a developer could know, at a glance, what a memory address held. That constraint no longer exists in any modern language.

**Options:**

1. **Systems Hungarian notation** — prefix the mechanical data type: `strName`, `iCount`, `pBuffer`, `lpData`, `IUserService` for an interface.
2. **Type-agnostic names** — rely entirely on the type system: `name`, `count`, `buffer`, `userService`.

**Trade-offs:**

[Consensus] In any modern statically or gradually typed language — Go, Rust, Java, C#, TypeScript, Kotlin, Swift — the compiler already tracks the type. Encoding it in the name duplicates information the toolchain provides more reliably. When the type changes, the name doesn't follow automatically: `intAge` becomes a float, `strName` becomes a rich object, `pUser` becomes a value. The name is now a lie. This is the liar variable: a convention that started as redundant and degraded into actively misleading.

Systems Hungarian is a solution to a problem that modern languages solved at the language level. Carrying it forward is cargo-culting.

**Common failure modes:** A loosely typed codebase adopts Hungarian notation. A `intAge` field is later updated to hold a floating-point value. No one renames it because the cost is high and the wrong name isn't breaking anything yet. A new engineer reads `intAge`, infers integer arithmetic is safe, and introduces a truncation bug.

**Example:** The Windows API prefixes (`lp`, `str`, `int`, `dw`) were genuinely necessary when the API was designed. C provided no way to inspect types without reading the declaration, and early editors provided no assistance. The same notation in a Rust or TypeScript codebase in the current decade provides nothing the compiler doesn't already surface — and costs accuracy every time a type changes.

---

### Use Predicate Prefixes for Boolean Identifiers

**What it is:** The convention of prefixing boolean variables and boolean-returning functions with `is`, `has`, `can`, or `should`.

**Why it exists:** This is one of the few naming conventions that carries real disambiguating value rather than pure style. A boolean and an object can have the same domain name — `permission` could be a boolean flag or a `Permission` object. The prefix removes the ambiguity without encoding memory layout.

**Options:**

1. **Semantic predicate naming** — `isActive`, `hasPermission`, `canRetry`, `shouldPersist`.
2. **Type-agnostic naming** — `active`, `permission`, `retry`, `persist`.

**Trade-offs:**

[Strong Recommendation] Use predicate prefixes for booleans. They do what Hungarian notation claimed to do but failed at: they communicate semantic state, not memory layout. `isActive` tells the reader this is a binary condition, not an enum or an object. The prefix is stable across refactors — the boolean meaning doesn't change even if the underlying representation does.

The difference between `if (ready)` and `if (isReady)` is not stylistic. The second version tells the reader the variable is a boolean condition being tested, not an object being used as a truthy value. In languages where objects are truthy, this distinction prevents misreading.

**Common failure modes:** Boolean variables named `enabled`, `permission`, or `valid` leave readers uncertain whether they are testing a flag or checking an object for truthiness. In conditional logic, ambiguous names require context to interpret — context the reader has to stop and fetch rather than reading the line at face value.

---

### Match Name Length to Scope

**What it is:** Identifier length should scale with how much context the surrounding code already provides.

**Why it exists:** A name exists to distinguish one thing from everything else the reader might have in mind. Inside a three-line loop, almost no distinction is needed. Across an entire package, substantial context must be encoded. A 30-character name in a tight mathematical loop is visual noise that obscures the algorithm; a single-letter variable at package scope is a collision waiting to happen.

**Options:**

1. **Scope-proportional naming** — short names (`i`, `err`, `db`) in small local scopes; longer descriptive names at broader visibility.
2. **Uniform verbosity** — fully descriptive identifiers everywhere regardless of scope.
3. **Aggressive abbreviation** — short names everywhere regardless of scope.

**Trade-offs:**

[Strong Recommendation] **Scope-proportional naming** is correct. The amount of descriptive text a name needs to carry is inversely proportional to how obvious the meaning is from immediate context. `i` in a three-line loop is unambiguous. `i` as a package-level variable is a disaster.

**Uniform verbosity** clutters tight algorithmic code. A loop variable named `currentArrayIndex` forces the reader to parse a noun phrase on every iteration instead of following the arithmetic.

**Aggressive abbreviation** at broad scope creates the global abbreviation failure: a developer names a package-level variable `c` for `Client`. A second developer introduces a `Cache`. The short name is now ambiguous and the collision requires a rename.

**Common failure modes:** A developer applies one rule everywhere. Either every local variable becomes `currentIterationIndex` and every loop reads like a legal document, or every exported function is a cryptic two-letter abbreviation. The failure in both directions is treating scope as irrelevant.

**Example:** Idiomatic Go uses `err` for local error checking, `i` for loop counters, and `db` for a local database handle — deliberately short because the scope is tight and the context is obvious. At package scope, exported types and functions get fully descriptive names. This is a considered convention, not laziness; Go's own documentation explains the rationale. Java codebases typically favor verbosity even at local scope, which is why Java developers rely heavily on IDE autocompletion just to write a loop. Neither is wrong — they optimize for different assumptions about how much context a reader holds.

---

### Prefer a Better Name Over an Explanatory Comment

**What it is:** When a block of code requires a comment to explain what it does, the first question is whether extracting it into a well-named function eliminates the need for the comment entirely.

**Why it exists:** A comment describes behavior. A function name *becomes* the behavior's description at every call site. A reader calling `recalculateCreditLimit()` doesn't need a comment explaining what is about to happen; the name already told them. The comment would be redundant, and redundant documentation rots.

This connection is the starting point for a longer treatment in [Ch 30](ch30-comments-what-to-comment-what-not-to.md), which covers the full distinction between comments that explain *what* and comments that explain *why*.

**Common failure modes:** A long function accumulates inline section headers:

```
// Validate input
// Build request
// Retry failed operations
// Persist result
```

Each heading describes a logical operation that could be a named function. The comments are a signal that the naming work wasn't done — and that the comments will drift when the code changes, while a function name would have been refactored alongside it.

---

### Why Smart Engineers Disagree: Style vs. Substance

The persistent engineering debates about naming are almost never about substance. `camelCase` vs. `snake_case`, `PascalCase` for types vs. interfaces — these are pure style choices. They carry no correctness value, prevent no bugs, and hide no information. The human brain adapts to any consistent formatting in a matter of days.

The only failure mode from casing choices is *inconsistency*. A codebase that arbitrarily mixes `camelCase` and `snake_case` forces developers to memorize the convention on a per-function basis, adding cognitive load where none needs to exist. The correct response is to pick the idiomatic convention for the language, enforce it with a linter on day one, and ban further debate. The linter doesn't care which you chose; it cares that you chose one.

The real disagreement among experienced engineers is about calibration: how much information should a name carry? One camp prefers long, fully explicit identifiers everywhere, arguing that explicitness reduces the chance of misreading. Another prefers concise names when surrounding context already supplies the missing information, arguing that excess length obscures structure. Both camps agree on the goal — make the surrounding code easy to read — and disagree only on how much of that work the name itself should do versus how much the reader's context window already carries.
