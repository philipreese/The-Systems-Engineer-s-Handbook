# Ch 33 — When to Write Unsafe or Low-Level Code

**Prerequisites:** [Cost Models and Mechanical Sympathy](../part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md) (the hardware reasoning that justifies unsafe code when it is justified), [FFI and Native Binding Design](../part03-api-design/ch26-ffi-and-native-binding-design.md) (the boundary-wrapper discipline this chapter reuses at the single-language level)

**New vocabulary introduced:** undefined behavior, safety escape hatch

**Key takeaways:**
- Unsafe code transfers correctness responsibility from the compiler to the engineer. That transfer carries a permanent maintenance cost; it must be purchased with demonstrated need.
- Legitimate reasons to write unsafe code are narrow: a measured, profiled performance bottleneck where the safe abstraction's overhead is the proven cause; the FFI boundary (covered in [Ch 26](../part03-api-design/ch26-ffi-and-native-binding-design.md)), where contact with unsafe code is unavoidable; implementing a foundational component that everyone else uses safely.
- Unsafe code must be immediately wrapped by a safe public API. The thin-binding, thick-wrapper structure from [Ch 26](../part03-api-design/ch26-ffi-and-native-binding-design.md) applies directly here: concentrate unavoidable unsafety into a small, auditable core while allowing the rest of the system to remain safe.
- Every unsafe block depends on invariants the compiler can no longer verify. Document them explicitly — they become part of the code's correctness contract.
- Unsafe code deserves more review, more testing, and narrower ownership than ordinary application code. The less the compiler verifies, the more engineering discipline must compensate.

---

Most software should be written entirely within its language's safety guarantees. Memory safety, type safety, bounds checking, and managed runtimes exist because they eliminate entire classes of defects — not as obstacles but as structural guarantees the compiler enforces on the programmer's behalf. This chapter is about the narrow set of circumstances in which stepping outside those guarantees is justified, and the discipline required when it is.

Rust's `unsafe` keyword is the clearest available model: an `unsafe` block suspends the compiler's memory safety verification for exactly that block, leaving the rest of the file fully checked. This is a deliberate design choice — a bounded, auditable escape from safety rather than a global setting. C and C++ take the inverse approach: unsafe by default everywhere, with the programmer bearing continuous responsibility. Most managed languages (Java, Python, C#, Go) occupy the safe-by-default position and offer limited mechanisms to escape it when necessary.

---

### Justify Unsafe Code with Measurement, Not Anticipation

**What it is:** The requirement that performance-motivated unsafe code be preceded by profiling evidence that identifies the safe abstraction's overhead as the actual bottleneck.

**Why it exists:** Safe abstractions have a physical cost. Array bounds checks emit CPU instructions. Garbage collectors introduce pause and overhead. Managed type systems constrain memory layout. These costs are real — but they are often not the dominant cost in a given system. Assuming they are, and bypassing them based on that assumption, introduces undefined behavior in exchange for a speedup that a profiler would have shown was unavailable in the first place.

**Options:**

1. **Safe implementation by default** — accept the language's guarantees and the performance they imply.
2. **Compiler-assisted optimization** — rely on the optimizer to prove invariants and eliminate safety checks through loop unrolling, vectorization, and constant propagation.
3. **Explicit unsafe escape after profiling** — bypass safety checks manually after measurement confirms they are the primary bottleneck under production conditions.
4. **Speculative unsafe optimization** — bypass safety checks based on assumption rather than evidence.

**Trade-offs:**

[Strong Recommendation] **Safe implementation by default** is correct for all code except the narrow cases described in "When to choose each option" below. The compiler's guarantees are not expensive in most code paths; they are background infrastructure.

**Compiler-assisted optimization** is worth attempting before any unsafe escape: modern compilers can often prove that an index is always in bounds and eliminate the check entirely. This approach is fragile — a benign refactor can silently cause the optimizer to stop applying the optimization — but it preserves safety guarantees and should be tried first.

**Explicit unsafe escape after profiling** is the correct path when measurement identifies the safe abstraction's overhead as the proven bottleneck. The profiling requirement is not a formality; it determines whether the unsafe code purchases anything real.

**Speculative unsafe optimization** — introducing unsafe code before profiling confirms the bottleneck — is always wrong. It permanently increases maintenance cost and introduces undefined behavior risk in exchange for a performance gain that was assumed, not measured.

**When to choose each option:** Choose an explicit unsafe escape when profiling under production conditions demonstrates that the safety mechanism — not I/O, not algorithmic complexity, not serialization, not network latency — is the primary bottleneck. The three legitimate triggers are: a measured performance bottleneck attributable to a specific safe abstraction; the FFI boundary ([Ch 26](../part03-api-design/ch26-ffi-and-native-binding-design.md)), where interaction with native code is unavoidable; and the implementation of a foundational component that exposes a safe interface to all its callers — someone must write the one well-tested unsafe core that a standard collection or allocator is built on.

**Common failure modes:**

*The Speculative Optimization Vector.* An engineer identifies an ingestion loop and assumes the bounds check on each array index is "too slow." Without capturing a baseline profile, they replace the safe index operation with an unchecked pointer offset. The unsafe path yields a statistically insignificant speedup — the loop was I/O bound, not CPU bound. The bounds check was not the bottleneck. The unsafe code now introduces a potential buffer overflow: a future caller with an unexpected input size gets memory corruption instead of a recoverable bounds error.

**Example:** High-throughput serialization engines like SIMD-json and custom binary protocol parsers routinely employ unsafe memory access. At 50 GB/s ingestion rates, bounds-checking every byte of an incoming network stream can consume a measurable fraction of the available CPU budget. The justification is valid in those systems — but it followed profiling. The engineer identified the specific instruction sequence being emitted for bounds checks, verified it was dominating cycle counts, and only then introduced the unsafe equivalent.

---

### Minimize the Unsafe Surface and Wrap It Immediately

**What it is:** The structural rule that unsafe code must be confined to the smallest possible region and immediately enclosed by a safe public interface that prevents unsafe invariants from leaking to callers.

**Why it exists:** Unsafety is contagious. A raw pointer, an unverified memory layout assumption, or an unchecked index exposed through a public interface means every caller must reason about low-level correctness. The undefined behavior blast radius — the scope of memory corruption a single bug can cause — grows with every line of code that operates on unverified data. Encapsulating unsafe code behind a safe boundary constrains that blast radius to an auditable core.

**Options:**

1. **Scattered inline unsafety** — place unsafe operations directly inside application-layer functions wherever they are needed.
2. **Hermetic safe packaging** — confine all unsafe operations to a dedicated, narrowly scoped module or type, exposing only safe, idiomatic methods to consumers.

**Trade-offs:**

[Strong Recommendation] **Hermetic safe packaging** is the only acceptable structure for unsafe code. This is the same thin-binding, thick-wrapper architecture established for FFI boundaries in [Ch 26](../part03-api-design/ch26-ffi-and-native-binding-design.md), applied within a single language. The unsafe implementation lives in one place; the safe API wraps it; every other part of the codebase calls the safe API. An engineer auditing the unsafe code knows exactly where to look. A bug in the unsafe region cannot propagate beyond the boundary because the boundary prevents callers from accessing the raw internals.

[Consensus] **Scattered inline unsafety** is not a trade-off with a legitimate position — it is an anti-pattern. Spreading unsafe operations through application-layer functions turns the entire codebase into an audit target. Memory corruption bugs become impossible to localize because the corrupted operation could have originated from any of dozens of scattered sites.

**Common failure modes:**

*The Leaky Pointer Contract.* A C# module uses an `unsafe` block to pin a managed array and extract its raw memory address, then returns that pointer directly to an application-layer service to avoid a copy operation. The application-layer service stores the pointer. Later, the originating function exits, the pinning lapses, and the garbage collector reclaims and reallocates the underlying memory. The application-layer service now holds a dangling pointer. The next read returns arbitrary memory contents. The corruption manifests as corrupted fields in an unrelated data structure, far from the original unsafe code, hours after the actual fault.

**Example:** Rust's `Vec<T>` is the clearest available demonstration that hermetic safe packaging works. Its implementation is built on raw pointers, manual reallocation, and unsafe memory management. Its public interface — `push`, `pop`, `len`, `iter` — is entirely safe. A caller can use `Vec` arbitrarily without ever writing a single `unsafe` block, because all the unsafe reasoning is contained in one well-tested, carefully reviewed core. The standard library pattern is not a special case; it is the template.

---

### Document Every Invariant the Compiler Can No Longer Verify

**What it is:** The requirement to record explicitly, adjacent to each unsafe block, the exact conditions that make the operation safe — the invariants the compiler would have verified automatically and no longer can.

**Why it exists:** Inside safe code, the compiler is the source of truth for memory and type correctness. When an unsafe block disables part of that verification, the engineer assumes the compiler's role. Without a written explanation of why the operation is safe, a future engineer modifying adjacent code has no way to know which assumptions must be preserved and which code changes would silently violate them. The invariant documentation is not a courtesy — it is part of the code's correctness proof.

**Options:**

1. **Explicit invariant attestation** — document the precise memory, layout, and ownership conditions that must hold for the unsafe operation to be correct, directly above the unsafe block.
2. **Implicit technical trust** — rely on the original engineer's competence and on code review to preserve invariants over time.
3. **Safe enforcement at the boundary** — validate inputs and state through ordinary safe code before entering the unsafe region, mechanically preventing invalid entry where possible.

**Trade-offs:**

[Strong Recommendation] **Explicit invariant attestation** is required for every unsafe block. An engineer writing unsafe code should be able to complete the statement: "This operation is safe because..." — clearly, specifically, and in writing. If they cannot, the code is not ready.

**Implicit technical trust** degrades predictably. The original engineer is not the last person who will modify adjacent code. Within months, someone who did not write the unsafe block will refactor a calling function and unknowingly violate the memory layout assumption that made the unsafe operation correct. The resulting bug appears far from the original change, under specific conditions, and intermittently.

**Safe enforcement at the boundary** — validating inputs before entering an unsafe region — is the correct complement to invariant documentation, not a replacement for it. Not every invariant can be mechanically checked: a pointer's continued validity, an alignment guarantee inherited from an upstream allocator, an assumption about the absence of concurrent modification. These require documentation even when partial validation at the boundary is also present.

**Common failure modes:**

*The Untested Padding Assumption.* An engineer writes a high-speed parsing routine using unsafe memory casts, assuming input buffers are always padded to 16-byte boundaries — "because that's how the upstream service currently works." No invariant comment documents this precondition. Two years later, a new upstream client sends a 7-byte buffer. The unsafe routine performs an out-of-bounds read, reaches an unmapped memory page, and crashes the process with a segmentation fault. The precondition was real and necessary; it was simply invisible.

**Example:** Idiomatic Rust community practice mandates a `// SAFETY:` comment above every `unsafe` block, documenting the exact reasons the operation cannot violate memory hygiene:

```rust
// SAFETY: The caller guarantees that `index` is less than the allocated
// capacity. This precondition is verified by the public `get_unchecked`
// wrapper, which panics on out-of-bounds access in debug builds and
// performs a boundary check at the safe API layer (line 42).
unsafe {
    return *self.ptr.add(index);
}
```

The comment is not stylistic convention — it is a human proof of correctness that replaces the compiler's automated verification. It gives reviewers a precise checklist. It tells future engineers exactly which assumptions must survive any refactor of the surrounding code.

---

### Hold Unsafe Code to a Higher Engineering Standard

**What it is:** The explicit elevation of review, testing, documentation, and ownership requirements for any code that bypasses a language's safety guarantees.

**Why it exists:** Bugs in unsafe code have consequences that safe-code bugs do not. An out-of-bounds read in safe code produces a recoverable exception or panic — a bounded, detectable failure. An out-of-bounds read in unsafe code produces undefined behavior: corrupted memory, incorrect values written to unrelated data structures, or exploitable security vulnerabilities. The consequences may appear far from the original fault, under specific conditions, long after the bug was introduced. Ordinary engineering process is not calibrated for these failure characteristics.

**Options:**

1. **Ordinary standards** — apply the same review and testing processes to unsafe code as to any other code.
2. **Elevated standards** — apply stricter review processes, broader test coverage, mandatory fuzzing, and narrower ownership to unsafe code specifically.

**Trade-offs:**

[Strong Recommendation] **Elevated standards** for all unsafe code. Unsafe implementations should receive: mandatory expert code review beyond the standard reviewer threshold; fuzz testing targeting the safe API boundary and the invariant conditions documented in safety comments; narrower ownership, with a limited set of engineers authorized to modify the unsafe core; and change management that requires documenting what invariant evidence justifies any modification. The additional process overhead is the price of stepping outside the compiler's verification. When the compiler stops checking, engineering process must compensate.

**Common failure modes:** Unsafe code proliferates through a repository one small addition at a time — "this is just a helper function," "this is obviously safe," "we'll clean this up later." Reviewers see enough unsafe blocks that they stop treating each one as exceptional. The explicit `unsafe` keyword, which was meant to draw attention, becomes background noise. The blast radius of any single bug, formerly constrained to one auditable module, now spans the entire codebase. Rust's explicit safety block is only useful as a flag if the team actually treats it as one.

**Example:** The Linux kernel treats its unsafe-by-nature C code with processes that safe-language codebases rarely apply: mandatory co-maintainer review, architecture-level mailing list discussion for changes to core subsystems, and lock-ordering documentation that is maintained with the same rigor as the code itself. The elevated process exists because the consequences of a bug in kernel memory management are catastrophic and the compiler provides no automatic safety net.

---

### Why Smart Engineers Disagree: Language Purity vs. Pragmatic Hardware Access

The genuine tension in unsafe code decisions separates engineers who treat safety guarantees as near-inviolable from those who treat hardware access as a first-class engineering tool.

Language purists argue that a single unsafe block weakens the correctness argument for the entire system. If one component relies on manually verified invariants instead of compiler-enforced ones, the mathematical safety guarantee of the codebase is no longer uniform. They will accept a 2x memory or latency penalty — even a significant one — to preserve the property that every correctness claim in the codebase is machine-checkable. From this perspective, the unsafe escape hatch exists for emergency use, not routine optimization.

Pragmatic systems engineers view this stance as an ideological position that hardware cannot always afford. At the bottom of the stack, everything is unsafe: syscalls, memory-mapped hardware registers, and DMA transfers do not have type checkers. Refusing a well-contained, thoroughly audited unsafe optimization out of theoretical purity ignores real hardware budgets and the systems that depend on them.

The correct resolution is not a position on that spectrum but a shift in the question being asked. The purpose of a language's safety guarantees is not to make unsafe operations impossible — it is to make them visible, localized, and auditable. An `unsafe` block in Rust or C# does not disable safety verification for the file or the application; it draws a clear boundary around a micro-scoped zone. The systems engineer does not reject safety and does not reject hardware access. They use hermetic safe packaging to buy both: a narrow unsafe core, a safe public API, elevated engineering process, and explicit invariant documentation. The unsafe code is an implementation detail, not an architectural choice.

Where to draw the line on acceptable unsafe code is a calibration every team makes based on their domain's performance requirements, failure consequences, and engineering discipline. High-throughput networking, compression engines, and OS kernels sit at one end. Standard business logic applications should rarely if ever approach the question. The calibration matters less than having one explicitly, rather than drifting toward unsafe code without ever deciding to.
