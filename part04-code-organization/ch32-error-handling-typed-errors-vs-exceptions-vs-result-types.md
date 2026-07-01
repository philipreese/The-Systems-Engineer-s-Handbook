# Ch 32 — Error Handling: Typed Errors vs. Exceptions vs. Result Types

**Prerequisites:** [Reliability as a Design Principle](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) (specifically: fail-fast and the crash/corruption/wrong-answer taxonomy), [Error Handling Contracts](../part03-api-design/ch21-error-handling-contracts.md) (the wire-level counterpart to this chapter's in-process mechanisms)

**New vocabulary introduced:** invisible control flow, error-as-value, operational failure, exceptional failure

**Key takeaways:**
- Unchecked exceptions introduce invisible control flow: any function invocation can silently exit the call stack multiple frames up, and nothing in the signature warns the caller. Error-as-value makes every failure path visible in the type signature and explicit at each call site.
- Neither cost disappears. Exceptions impose invisible control flow; error-as-value imposes boilerplate. Trade them deliberately, matching the mechanism to the nature of the failure.
- Use error-as-value for operational failures — missing file, timeout, invalid input — that the immediate caller is realistically expected to handle.
- Use exceptions or fail-fast termination for exceptional conditions — programming bugs, violated invariants, unrecoverable environmental states — where local recovery would be wrong.
- Checked exceptions are a widely acknowledged historical misstep. Avoid them in new designs.
- Rust's `?` operator demonstrates that explicit propagation and readable success paths are not mutually exclusive; they require language support, not a fundamental trade-off.

---

[Ch 21](../part03-api-design/ch21-error-handling-contracts.md) covers how failures are serialized and communicated across a network boundary. [Ch 07](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) established the system-level taxonomy: crash failures, corruption failures, and wrong-answer failures. This chapter is the in-process layer between them — how failure is represented and propagated through function calls within a single codebase, before it ever reaches an API boundary.

The choice of mechanism is not cosmetic. It determines whether a caller knows a failure is possible, whether the compiler can enforce acknowledgment, and whether an engineer reading the code can trace every exit point without opening external dependency files.

---

### Choose the Error Propagation Paradigm

**What it is:** The foundational mechanism a codebase uses to route failures from the point where they occur to the point where they are handled.

**Why it exists:** Errors arise in low-level I/O and validation but are resolved by high-level policy. The propagation mechanism defines the cognitive and computational path an error takes through those layers — and whether that path is visible or hidden.

**Options:**

1. **Unchecked exceptions** — any function can throw without declaring it in its signature. The call stack unwinds automatically until a matching catch block is found. Python, JavaScript, and Java's `RuntimeException` follow this model.
2. **Checked exceptions** — the compiler requires a function that can throw to either declare the exception in a `throws` clause or handle it locally. Java's original exception hierarchy was built on this model.
3. **Error-as-value** — failure is returned as an ordinary value through the function's standard return channel. The function signature explicitly declares the possibility of failure; the type system tracks whether callers handle it. Go's `(result, error)` return convention and Rust's `Result<T, E>` are its clearest expressions.

**Trade-offs:**

[Strong Recommendation] **Error-as-value** is the correct default for production systems engineering. Every call site is an explicit acknowledgment that the operation can fail. An engineer auditing a production failure or reviewing a diff can trace every failure path without opening external files. Each `if err != nil` is attributable in git history and greppable by static analysis. The cost — boilerplate — is real. Go's canonical pattern:

```go
result, err := doSomething()
if err != nil {
    return nil, err
}
```

is the community's most cited complaint about the language. The designers accepted that complaint deliberately: the explicitness is the point. Go is not the first to make this choice — C's `errno` and integer return codes were error-as-value without enforcement, with the predictable result that developers routinely ignored return values including from `close()` and `printf()`. Go and Rust drew the lesson and added type-system enforcement.

Rust refines the model further with the `?` operator, which propagates a `Result::Err` to the caller in a single character while preserving the explicit contract in the function signature:

```rust
fn read_username(path: &str) -> Result<String, io::Error> {
    let mut file = File::open(path)?;
    let mut s = String::new();
    file.read_to_string(&mut s)?;
    Ok(s)
}
```

The failure path is fully visible in the return type. The boilerplate is gone. `?` demonstrates that the boilerplate objection to error-as-value is a language design problem, not an inherent trade-off between explicitness and readability.

[Consensus] **Unchecked exceptions** are acceptable for high-level application code, scripting, and rapid prototyping where crashing a request handler or script process is a cheap and adequate isolation strategy. They are not appropriate for systems where hidden failure paths carry meaningful operational risk.

[Strong Recommendation] **Checked exceptions** should be avoided in new designs. Java's checked exception experiment was motivated correctly — preventing ignored failures through compiler enforcement — but produced cascading signature pollution and a specific anti-pattern (the catch-and-ignore block, covered below). Major Java frameworks including Spring bypassed checked exceptions entirely by wrapping internal failures in unchecked `RuntimeException`. No major language designed after Java has adopted checked exceptions as a primary mechanism. The lesson has been absorbed by the industry.

**When to choose each:** Use error-as-value for any system where failure paths must be auditable, where callers are expected to respond to failures specifically, or where reliability is an explicit design constraint. Use unchecked exceptions in scripting, prototyping, or application layers where a crashed process is the acceptable failure mode. Avoid checked exceptions in greenfield work.

**Common failure modes:**

*The Ghost Crash.* In a Node.js service, an engineer invokes a third-party library's data transformation function. Nothing in the signature or documentation indicates the function can throw a socket error under load. In production, an unexpected network state causes the exception to propagate past every request handler into the process's uncaught exception handler, terminating the server instance. Upstream callers all fail simultaneously. The failure was not unrecoverable — the connection could have been retried — but no caller along the stack knew one was possible.

*The Catch-and-Ignore Block.* Java's checked exception mechanism forces a caller to either declare or handle every checked exception. Faced with a `throws IOException` on a function that realistically can't fail in context, or simply in a hurry, a developer writes:

```java
try {
    readConfig();
} catch (IOException e) {
    // TODO: handle this
}
```

The compiler is satisfied. When `readConfig()` does fail in production, the exception disappears silently. The compiler's requirement was met; no understanding was gained.

**Example:** C established error-as-value. Go institutionalized it with a first-class `error` return type and cultural convention. Rust formalized it with `Result<T, E>` and removed the syntax objection with `?`. Each represents the same underlying philosophy with progressively better tooling around the verbosity cost.

---

### Match the Mechanism to the Nature of the Failure

**What it is:** The distinction between failures that are expected parts of correct execution and failures that indicate the program is in a state it cannot safely continue from.

**Why it exists:** Treating every failure as exceptional — throwing an exception for a missing file, for a validation error, for a timeout — forces callers to handle exception semantics for outcomes they were fully prepared to manage through normal control flow. Treating every failure as operational — catching a null-dereference or out-of-memory error and returning it as a value — means continuing execution in a state the program was not designed to survive.

**Options:**

1. **Return operational failures as values** — file not found, timeout, invalid input — expected anomalies during correct execution that the immediate caller is realistically expected to handle.
2. **Reserve exceptions and panics for exceptional conditions** — out-of-memory, null dereference in code that asserted non-null, an array access beyond its bounds — unrecoverable violations of invariants that indicate a programming bug or environmental collapse.
3. **Uniform recovery** — catch all failures, including invariant violations, inside a top-level handler to keep the process alive.

**Trade-offs:**

[Consensus] **Return operational failures as values** and **reserve exceptions and panics for exceptional conditions.** Go encodes this directly: `error` for operational failures; `panic` for invariant violations. Rust encodes it as `Result<T, E>` for expected failure; `panic!` for unrecoverable ones. The two categories deserve different mechanisms.

**Uniform recovery** — a top-level `catch (Exception e)` or `recover()` that intercepts everything — is the failure mode described in [Ch 07](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) as the wrong-answer failure: the system continues operating on corrupted state, producing incorrect results with no visible indication that anything is wrong. This is strictly worse than a crash.

**When to choose each:** If the immediate caller is realistically expected to respond to the failure — retry, substitute a default, report an error to the user — return it as a value. If the failure indicates the program's own code has a bug, or that the runtime environment has collapsed beyond recovery, terminate through a panic or exception and let the external orchestrator (Kubernetes, systemd, a supervisor process) restart from a clean state.

**Common failure modes:**

*The Undefined State Zombie.* A service wraps its entire HTTP request handler in a `try { ... } catch (err) { res.status(500).send() }`. A request triggers a `TypeError: Cannot read property 'balance' of undefined` deep in the payment state manager — a programming bug. The catch block intercepts it, prevents the crash, and keeps the process running. The bug has left the payment state in an undefined condition. Subsequent unrelated requests operate on that corrupted state, producing incorrect account balances. The service is alive and wrong. This is the worst failure mode in the [Ch 07](../part01-systems-thinking/ch07-reliability-as-a-design-principle.md) taxonomy: silent, self-reinforcing, and masked by apparent health.

**Example:** Array index out-of-bounds access indicates a programming error — the code made an incorrect assumption about its own state. Failing to open a user-selected file because it doesn't exist is a routine operational event the file-opening call was designed to report. These failures are categorically different and deserve categorically different handling. The type of failure, not the severity of its consequence, governs the mechanism.

---

### Accept Language Boilerplate or Use Language Features — Don't Build Frameworks

**What it is:** The decision about how to reduce the syntactic cost of error-as-value propagation.

**Why it exists:** The boilerplate cost of explicit error forwarding is real, repetitive, and tempts engineers to abstract it away through custom machinery. The result is almost always worse than the original verbosity.

**Options:**

1. **Accept the language idiom** — write explicit error-forwarding at each call site.
2. **Use language-native propagation syntax** — Rust's `?` operator; language-idiomatic patterns where available.
3. **Build a custom propagation framework** — centralized error channels, bespoke result monads, higher-order error-threading utilities.

**Trade-offs:**

[Strong Recommendation] **Accept the language idiom** when it is the standard. Go's repeated `if err != nil` is a conscious deliberate decision by Go's designers, made with full awareness of the community's complaint. Eliminating it through a custom abstraction trades a minor readability cost for a significant auditability cost — an engineer unfamiliar with the custom machinery cannot read any code that uses it without first understanding the abstraction.

**Language-native propagation syntax** is the correct upgrade path when the language provides it. Rust's `?` is right because it preserves the explicit contract (`Result<T, E>` in the return type) while removing the forwarding ceremony. The failure remains visible; only the syntax is abbreviated. If the language does not provide this — and Go as of this writing does not — the correct response is to accept the idiom, not to build a substitute.

**Custom frameworks** are the wrong answer. A bespoke `Result` monad, a `Must(value, err)` helper, or a higher-order function that threads errors through callbacks all introduce a layer that an engineer must understand before being able to read code that uses it. The original boilerplate was at least the language standard.

**When to choose each:** Use the language idiom as the default. Adopt language-native syntax when the language provides it natively. Never build a custom error-propagation framework unless the team is writing infrastructure consumed by thousands of engineers who will all be trained on the abstraction — in which case the abstraction has become the language, and should be held to that standard.

**Common failure modes:**

*The DIY Panic Wrapper.* A team frustrated with Go's verbosity introduces a `Must(value, error)` helper that panics on any non-nil error, eliminating all `if err != nil` forwarding. Call sites become cleaner. Six months later, a routine operational failure — a timeout during a non-critical metadata lookup — causes the service to crash because `Must` was applied to a function the author assumed could not fail in production. The helper conflated operational and exceptional failures in its design, and the consequences arrived at runtime.

**Example:** Spring's shift from checked exceptions to unchecked `RuntimeException` was not a shift toward ignoring errors — it was a recognition that the checked exception mechanism was the wrong carrier for the principle of explicit failure acknowledgment. The principle survived; the mechanism changed. Rust's `?` operator is the same lesson applied to error-as-value: the principle (explicit failure in the type signature) survived; the boilerplate (manual `match` on `Result`) changed.

---

### Why Smart Engineers Disagree: The Author vs. the Maintainer

The argument about error-handling paradigms is at its core an argument about who the code is being optimized for.

Engineers who favor exceptions argue that code is most readable when the success path dominates the visual structure. Interleaving error-checking conditionals with domain logic forces the reader to context-switch constantly, making it harder to reason about the primary algorithm. From this perspective, exceptions do the right thing: primary logic flows linearly, exceptional paths are handled separately, and the happy path reads like the requirements document it's meant to implement.

Engineers who favor error-as-value argue that failure paths are not incidental noise but core program logic. In systems engineering, the error-handling code often constitutes the majority of the codebase — resource cleanup, partial failure recovery, retry decisions, user-facing error messages. Hiding those paths behind invisible control flow means a reader cannot verify correctness without knowing which of the dozens of function invocations in any given block might silently exit multiple frames up.

The practical synthesis: implicit exceptions favor the author of the code, who is writing the happy path. Explicit values favor the maintainer, who is auditing a production incident at 2 a.m. or reviewing a diff for safety before merging it. The author writes the code once. The maintainer reads it many times under pressure.

The direction of language design has moved toward explicit values. Go chose verbosity deliberately. Rust proved the boilerplate objection is a solvable language problem. The industry's movement away from checked exceptions was not a retreat to unchecked exceptions; it was a recognition that the type system — not the exception mechanism — is the right place to make failures visible. The argument between exception-first and value-first paradigms will persist as long as exception-based languages remain dominant, but the architecture of new systems increasingly reflects the value-first answer.
