# Part IV — Chapter Specifications

Pre-filled special instructions for each Part IV chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part IV: Part I, especially Ch 02 (complexity) and Ch 04 (abstraction and
information hiding). Several chapters also build directly on Part II and Part III content
that was deliberately deferred here — see the forward references below.

Two forward references already exist and must be honored here:
- Part I Ch 03, Ch 04, Part II Ch 11, and Ch 12 all promised that file and module structure —
  specifically, how to physically organize the interfaces and implementations that dependency
  inversion and hexagonal architecture produce — is covered in Ch 27.
- Part III Ch 26 promised that the broader practice of writing unsafe or low-level code is
  covered in depth in Ch 33.

---

## Ch 27 — File and Module Structure

```
This chapter answers the question Part I Ch 03 and Part II Ch 11/Ch 12 deliberately deferred:
once coupling, cohesion, and dependency direction are understood as principles, how do they
actually show up as folders and files in a real codebase? This is the physical, code-level
expression of those decisions — not a re-derivation of them.

Cover package-by-layer vs. package-by-feature as the central organizing decision. Package-by-
layer (a `controllers/`, `services/`, `repositories/` split) groups code by its technical role;
package-by-feature (an `orders/`, `billing/` split, each containing its own controller,
service, and repository) groups code by the domain concept it serves. Be explicit that
package-by-feature has won broad industry consensus for most application code, while
package-by-layer still has a place in framework and platform code with genuinely generic roles.

Cover where the ports and adapters from Ch 11 actually live as files: the core/domain package
contains the ports (interfaces) and is dependency-free; the adapters live in their own
packages, one per infrastructure technology, depending inward on the core but never the
reverse. This makes Ch 12's dependency rule mechanically enforceable by which package is
allowed to import which.

Cover circular dependency avoidance as a structural consequence of getting this wrong — two
packages that import each other are almost always evidence that a concept was split across a
boundary it should never have crossed.

Anchor in real systems: idiomatic Go's strong cultural rejection of package-by-layer (a
`controllers`/`models`/`utils` split is considered a known anti-pattern in the Go community,
documented repeatedly by Go's own maintainers), Rust's module system and explicit `pub use`
re-exports as a language-level enforcement of information hiding at the file level, a "god
package" (often literally named `common` or `utils`) accumulating unrelated code as the
canonical failure case.

Do NOT cover: coupling and cohesion as concepts (Part I, Ch 03), the dependency inversion
principle itself (Part II, Ch 12), the hexagonal/layered architecture patterns themselves
(Part II, Ch 11), naming conventions for the files and identifiers within this structure
(Ch 28), the finer-grained question of splitting a single concept across multiple files
(Ch 29). This chapter is about package- and module-level organization specifically.
```

---

## Ch 28 — Naming Conventions and When They Matter

```
Naming is the cheapest form of information hiding available — a good name communicates
intent without costing any indirection at all. This chapter separates naming that actually
carries information from naming convention that is pure style, with no correctness or
clarity benefit either way.

Cover intention-revealing names directly: a name should match the domain's own vocabulary
(the same ubiquitous-language principle behind bounded contexts in Part II, Ch 13) rather
than the implementation's internal structure. Cover the specific, common failure of Hungarian
notation and type-prefixing as a historical anti-pattern: genuinely useful in untyped C, where
the type wasn't visible anywhere else, and now redundant cargo-culting in any modern
statically or gradually typed language where the compiler already tracks the type.

Cover boolean and predicate naming (`is`/`has`/`can` prefixes) as one of the few conventions
that does carry real disambiguating value, distinct from arbitrary style choices like
camelCase vs. snake_case, which matter only for consistency within a codebase, never for
correctness.

Cover the direct link to Ch 30: a block of code that needs a comment to explain what it does
is frequently a sign that extracting it into one well-named function would have made the
comment unnecessary in the first place.

Anchor in real systems: the "intention-revealing names" principle from Extreme Programming
and the broader Clean Code tradition, idiomatic Go's deliberate convention of very short
variable names in small scopes (`i`, `err`) versus fuller names at package scope — a
considered convention, not sloppiness, that contrasts with Java's typical verbosity at every
scope, Hungarian notation's original C-era justification versus its modern misuse.

Do NOT cover: comments as an alternative communication mechanism (Ch 30, beyond the brief
connection above), abstraction design (Ch 31), file and module naming as an organizational
question (Ch 27 — this chapter is about naming the identifiers within that structure, not the
structure itself).
```

---

## Ch 29 — When to Split Files vs. Keep Together

```
Given that Ch 27 already established package- and module-level organization, this chapter is
about a finer-grained decision: should this one class, this one concept, live in its own file,
or alongside closely related code? This is a cohesion question (Part I, Ch 03) applied at the
smallest practical grain, and it's governed by the same Rule of Three (Part I, Ch 04) that
governs premature abstraction — don't split preemptively on a guess; split once the evidence
that two things are actually separate concerns is real.

Cover the "one class per file" convention (enforced by some ecosystems, like Java) against the
"file as a unit of related concepts" convention common in Go, Python, and JavaScript, where
several small, tightly related functions or types legitimately share one file.

Cover concrete signals for splitting: a file that's grown long enough to be hard to navigate,
or one that visibly contains multiple unrelated concerns with low cohesion between them. Cover
concrete signals for keeping together: small mutually dependent types that only make sense
read side by side, or a small state machine whose pieces lose meaning split apart.

Briefly cover test co-location (a test file living next to its source file vs. in a separate
parallel test tree) as a related instance of the same question, without going deep — this is
a Part V (Testing Strategy) concern at its core.

Anchor in real systems: the frontend "co-location" convention (a component, its styles, and
its test living in one folder) as a deliberate cohesion-first choice, contrasted with
traditional layered separation (all components in one folder, all styles in another); a
"god file" that accumulated unrelated logic over years as the canonical failure case.

Do NOT cover: package- and module-level structure (Ch 27 — this chapter assumes a module
boundary already exists and asks about file granularity within it), naming (Ch 28), whether
splitting code into a new file constitutes a new abstraction worth its cost (Ch 31 — these are
related but distinct decisions; splitting a file is not automatically the same as introducing
an abstraction boundary).
```

---

## Ch 30 — Comments: What to Comment, What Not To

```
A comment is a maintenance liability: it has to be kept in sync with code that changes, and
unlike the code itself, nothing enforces that it actually is. This chapter is about when that
ongoing cost is worth paying.

Cover the WHY-vs-WHAT distinction as the central test: a comment restating what the next line
of code already says is pure cost with no benefit. A comment capturing a hidden constraint, a
workaround for a specific external bug, or a non-obvious invariant the code depends on but
doesn't visibly enforce — something a careful reader of the code alone could not recover — is
the only category worth its maintenance cost.

Cover comment rot directly as the dominant failure mode: a comment that no longer matches the
code is worse than no comment, because it actively misleads a reader who has no way to know
it's stale without independently re-deriving the truth anyway.

Cover the distinction between inline implementation comments (aimed at the next maintainer,
local to the function) and docstrings or API-level documentation comments (aimed at a
consumer of the interface, part of its public contract) as genuinely different categories
with different obligations — the latter is closer to the API surface design discipline from
Part III, Ch 15 than to ordinary inline commentary.

Cover the relationship to naming directly: a block of code that needs a comment to explain
itself is frequently better served by extracting it into a well-named function (Ch 28),
eliminating the comment's maintenance burden entirely rather than paying it forever.

Anchor in real systems: the Linux kernel's comment discipline — comments reserved for
genuinely non-obvious things like lock-ordering requirements or hardware quirks, never for
restating what a line already says — Python's PEP 257 docstring convention as a deliberate,
separate standard for the public-contract category of comments.

Do NOT cover: API documentation in depth as its own discipline (Part VIII, Ch 64–68), naming
as the primary subject (Ch 28, beyond the direct connection above), Architecture Decision
Records as the place large-scale "why" decisions belong (Part VI, Ch 45) — a comment is local,
in-code why; an ADR is repository-level decision history, and the two should not be conflated.
```

---

## Ch 31 — When Abstractions Help vs. When They Obscure

```
Part I Ch 04 established that the wrong abstraction is worse than none, and Part II Ch 14
applied that test at the architectural-layer level. This chapter applies the same test one
level down: a single function, class, or small utility. The core test doesn't change — does
this abstraction hide a decision genuinely likely to change — but the failure mode at this
grain is distinct: indirection introduced for code that will only ever have one realistic
shape, cluttering local readability without buying any real flexibility.

Cover the specific code-level smells directly: an interface with exactly one implementation
and no credible second one on the horizon; a generic, heavily parameterized function used in
exactly one call site; a strategy-pattern object standing in for a choice that's actually
fixed at compile time and never varies at runtime.

Cover the distinction between abstraction for flexibility (swappability, the Ch 12 dependency-
inversion motivation) and abstraction for readability (a small wrapper that exists purely to
give a noisy or awkward call a clear, domain-meaningful name) — the second is often justified
even with only one implementation, because the payoff is local clarity, not future
substitutability, and it should be evaluated on those terms instead of being held to a
swappability bar it was never trying to clear.

Anchor in real systems: "FizzBuzzEnterpriseEdition" as the well-known parody artifact
illustrating needless abstraction taken to its logical extreme, Rob Pike's Go proverb "a
little copying is better than a little dependency" as a real, quotable industry position that
favors concrete, duplicated code over a premature shared interface, a one-line wrapper
function that exists solely to rename a clumsy third-party API call into domain vocabulary as
the positive, readability-motivated case.

Do NOT cover: architectural-layer abstraction decisions (Part II, Ch 14 — this chapter is
about the function/class grain, not the service or layer grain), the dependency inversion
principle itself (Part II, Ch 12), the definition of the wrong-abstraction failure mode itself
(Part I, Ch 04 — reference it, don't redefine it here).
```

---

## Ch 32 — Error Handling: Typed Errors vs. Exceptions vs. Result Types

```
Part III Ch 21 covered how an API communicates failure across the wire; Part I Ch 07 covered
the system-level philosophy of fail-fast and the crash/corruption/wrong-answer taxonomy. This
chapter is neither of those — it's the in-language mechanism a single codebase uses to
represent and propagate failure internally, before it ever reaches an API boundary.

Cover the three dominant paradigms explicitly: unchecked exceptions (Java's RuntimeException,
Python, JavaScript), where any function can throw and nothing in its signature warns a caller;
checked exceptions (Java's checked Exception), where the type system forces a caller to
acknowledge a possible failure; and error-as-value (Go's `(result, error)` return convention,
Rust's `Result<T, E>`), where failure is just another value the type system tracks like any
other return.

Cover the core trade-off honestly: exceptions keep the happy path free of error-checking
boilerplate at every call site, at the cost of invisible control flow — in an unchecked-
exception language, nothing in a function's signature tells a caller what it might throw.
Error-as-value makes every possible failure visible in the type signature and forces explicit
handling at the call site, at the real cost of repetitive boilerplate (Go's repeated
`if err != nil` is the canonical complaint).

Cover when each is the right tool: exceptions for genuinely exceptional, rare, unrecoverable-
at-this-layer conditions — out-of-memory, a violated invariant that indicates a programming
bug — which is the same territory as fail-fast (Part I, Ch 07). Error-as-value or Result types
for expected, routine failure that's part of normal control flow and that the immediate caller
is realistically expected to handle — a missing file, a timeout, invalid input.

Anchor in real systems: Go's explicit error-as-value philosophy and the community's own
long-running debate about its verbosity, Rust's `Result<T, E>` paired with the `?` operator as
a refinement that keeps error-as-value but removes most of the boilerplate cost through
syntax, Java's checked-exceptions experiment as a widely acknowledged historical misstep —
most modern Java frameworks (Spring included) deliberately moved back to unchecked exceptions
— and C's `errno`/return-code convention as error-as-value's direct ancestor.

Do NOT cover: wire-level API error response contracts (Part III, Ch 21 — this chapter is the
in-process mechanism, not what crosses a network boundary, though the relationship is worth
one sentence), system-level fail-fast and reliability philosophy (Part I, Ch 07 — already
covers the taxonomy this chapter's exception case draws on), retries and idempotency
(Part III, Ch 22).
```

---

## Ch 33 — When to Write Unsafe or Low-Level Code

```
Part III Ch 26 promised this chapter would cover the broader practice of writing unsafe or
low-level code, beyond the FFI boundary specifically. Most application code never needs manual
memory management or to step outside its language's safety guarantees; this chapter is about
recognizing the narrow set of situations where doing so is actually justified, and the
discipline required when it is.

Cover what "unsafe" or "low-level" means concretely across ecosystems: Rust's explicit
`unsafe` keyword as an auditable, narrowly scoped boundary; C and C++ as unsafe by default
everywhere; bypassing a managed language's guarantees (an `unsafe` block in C#, a Python C
extension) as the equivalent move in a normally safe environment.

Cover the legitimate reasons to go there, specifically: a measured, profiled performance
bottleneck where a safe abstraction's overhead is the proven cause, not a guessed one (this
is the mechanical sympathy argument from Part I, Ch 06, applied as a concrete justification
test); the FFI boundary itself (Part III, Ch 26), where contact with unsafe code is
unavoidable by construction; implementing a safe abstraction that everyone else will use
safely — someone has to write the one well-tested unsafe core that a standard collection type
or allocator is built on.

Cover the discipline required once the decision is made: minimizing the unsafe surface to the
smallest possible block, immediately wrapped by a safe public API — the same thin-binding,
thick-wrapper structure from Ch 26's "Why Smart Engineers Disagree" resolution, reused here at
the single-language level instead of the cross-language one — documenting the specific
invariant the unsafe code relies on that the compiler can no longer verify, and holding unsafe
code to a visibly higher bar of testing and review than ordinary code receives.

Anchor in real systems: Rust's `unsafe` keyword as the most explicit, auditable version of
this decision available in any mainstream language — an `unsafe` block doesn't disable safety
checking for the rest of the file, only for that block, a deliberate design choice worth
calling out directly — the standard pattern of an unsafe implementation hidden behind a safe
public API in most languages' own standard libraries, and a profiling-first example where a
bounds-check was measured, not assumed, to be the actual bottleneck before unsafe code was
introduced to remove it.

Do NOT cover: FFI mechanics themselves, already covered in depth (Part III, Ch 26), general
profiling methodology and the broader decision of when to optimize at all (Part XII, Ch 85–86
— this chapter is specifically about the unsafe/low-level decision, not optimization
decisions in general), concurrency-specific unsafe patterns (Part X).
```
