# Chapter 26 — FFI and Native Binding Design

**Prerequisites:** [Part I, Ch 04 — Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Part III, Ch 15 — API Surface Design: What to Expose, What to Hide](../part02-software-architecture/ch15-api-surface-design-expose-hide.md), [Ch 22 — Idempotency](ch22-idempotency.md). Specifically: information hiding, minimal surface area, and the contrast with network-level graceful degradation this chapter inverts.

**New vocabulary introduced:** Foreign Function Interface (FFI), Application Binary Interface (ABI)

**Key takeaways:**
- This chapter is a deliberate register change from the rest of Part III: not a network boundary, but a language boundary — code in one language calling directly into code compiled in another. There's no timeout, no retry, no graceful degradation. A mistake at a network boundary returns a `500`; a mistake at an FFI boundary can terminate the entire host process. It's the least forgiving API surface in this book.
- An API is a logical contract; an ABI is the physical one — exact memory layout, struct padding, calling convention. Two different compilers agreeing on source-level behavior is not the same as them agreeing on what bytes a struct actually contains.
- Memory ownership is the central design problem, because there's no shared garbage collector watching both sides. Exactly one side has to be responsible for freeing what crosses the boundary, and that responsibility has to be explicit, not assumed.
- Exceptions, panics, and stack unwinding are language runtime features, not hardware features, and none of them can safely cross into a different runtime. Error handling at this boundary degrades to primitives: return codes, out-parameters, or a thread-local error slot — never a thrown exception reaching across languages.

---

## ABI vs. API: The Binary Contract

**What it is:** The distinction between the logical contract a human reads (the API — function signatures, types) and the physical, compiled contract the CPU actually executes (the ABI — memory layout, struct padding, register usage, calling convention).

**Why it exists:** A CPU doesn't read source code; it reads memory addresses and registers. Within one language, the compiler silently guarantees caller and callee agree on memory layout. Across an FFI boundary, two entirely different compilers are involved, and nothing guarantees they agree on anything — if they don't agree on byte alignment, the callee reads garbage straight out of the registers.

**Options:**
1. **Language-specific ABI** — exposing a higher-level language's native, often unstable, ABI directly (C++, Rust)
2. **The C ABI** — routing all cross-boundary communication through the stable, universally understood C calling convention

**Trade-offs:**
- *Language-specific ABI:* allows passing genuinely rich types (`std::vector`, generic traits) — but it's fundamentally unstable; C++'s ABI shifts with compiler (GCC vs. MSVC) and even optimization flags, so unless caller and callee are built with the exact same toolchain, the boundary silently corrupts memory.
- *The C ABI:* mathematically stable and supported by essentially every language — but it strips the surface down to the lowest common denominator: raw pointers, contiguous byte arrays, primitive integers. Every high-level feature has to be left at the door.

**When to choose each:**
- *C ABI:* the default for any library meant to be consumed by more than one language.
- *Language-specific ABI:* never across a real FFI boundary, unless caller and callee are built from the same monorepo with the same toolchain — which means it isn't really a foreign boundary at all.

**Common failure modes:**
- **The field-reordering segfault:** a C struct gets a new boolean field inserted in the middle. For C consumers who recompile, this is a non-breaking source change. For a Python binding that dynamically links without recompiling, the byte layout has shifted underneath it — the library reads memory at the old offsets, misinterprets a pointer, and segfaults the entire process.

**Example:** The Rust standard library deliberately does not guarantee a stable Rust ABI — the compiler is free to reorder struct fields for padding optimization at will. Exposing a Rust library to Python or C requires `#[repr(C)]` on structs and `extern "C"` on functions, explicitly forcing Rust to abandon its own optimizations and obey the C ABI at the boundary. **[Consensus: route every cross-language boundary through the C ABI; nothing else is stable enough to build on]**

---

## Cross-Boundary Memory Ownership

**What it is:** The explicit rule for which side of an FFI boundary allocates memory, and — more importantly — which side is responsible for freeing it.

**Why it exists:** Memory allocators aren't universal. A pointer allocated by Rust's allocator can't be garbage-collected by the JVM; if a C library allocates a buffer and hands it to Python, Python's GC doesn't know its size and isn't allowed to call `free()` on it. Without an explicit rule, the result is either a permanent leak or a double free.

**Options:**
1. **Caller allocates** — the high-level language allocates a buffer and passes a pointer in; the native library fills it
2. **Callee allocates (opaque pointers)** — the native library allocates internally and returns an opaque pointer; the caller must later pass that exact pointer to a library-provided free function

**Trade-offs:**
- *Caller allocates:* keeps memory entirely inside the calling language's GC domain, eliminating cross-boundary leak risk outright — but the caller has to know the size upfront, which for dynamically sized data often costs two FFI calls (ask for the size, then provide the buffer).
- *Callee allocates:* fully preserves information hiding — the caller never has to know the struct's size or layout — but the caller now has to perfectly track the pointer's lifecycle; if its own GC drops the reference without calling the native free function first, that memory is gone for good.

**When to choose each:**
- *Caller allocates:* strings, flat byte buffers, predictably sized numeric arrays.
- *Callee allocates:* complex domain objects, state machines, connections — anything where size or layout shouldn't leak to the caller, paired with an explicit `LibraryName_Free()`.

**Common failure modes:**
- **The cross-allocator free:** a C++ library allocates an object and returns it; a consumer frees it with plain C `free()` instead of `delete`, or worse, with a different runtime's allocator entirely (a different `.dll`'s C runtime, say). Heap metadata corrupts silently, and the application keeps running on corrupted state for hours before it finally crashes somewhere unrelated.

**Example:** SQLite's C API is the reference case for explicit ownership. `sqlite3_prepare_v2()` allocates internal state and returns an opaque `sqlite3_stmt*` — since the caller never allocated it, the contract requires passing that exact pointer to `sqlite3_finalize()` when done. Python's `sqlite3` standard-library module simply wraps this: a Python object's `__del__` calls `sqlite3_finalize()` automatically, translating an explicit C ownership contract into something that looks like ordinary garbage collection from the Python side.

---

## Error Handling Across FFI Boundaries

**What it is:** The mechanical translation of failure across a boundary that cannot physically propagate exceptions, panics, or stack unwinding.

**Why it exists:** Exceptions are a language runtime feature, not a hardware one. If a C++ exception or a Rust `panic!` tries to unwind the stack across an `extern "C"` boundary into a runtime with completely different unwinding mechanics — Python, Node's V8 — the result is undefined behavior, and in practice, an immediate crash.

**Options:**
1. **Return codes + out-parameters** — the function returns a primitive status integer; the actual result is written into a pointer the caller provided
2. **Thread-local error state** — the function returns a null/sentinel value on failure, and the caller separately queries a thread-local function (`getLastError()`) for the details

**Trade-offs:**
- *Return codes + out-parameters:* mathematically safe and entirely stateless, supported by every language — but deeply unergonomic; calls can't be composed, and every single call site needs its own mutable out-pointers and an `if (res != 0)` check.
- *Thread-local error state:* lets the function signature return the actual data, which reads far more naturally — but it relies on hidden mutable state. Forget to check it, or make another FFI call before checking it, and the error is silently overwritten and gone.

**When to choose each:**
- *Return codes + out-parameters:* the universal, safe default for low-level native bindings.
- *Thread-local error state:* only when integrating deeply into a runtime that already enforces this pattern globally.

**Common failure modes:**
- **The cross-boundary panic:** a native math library written in Rust exposes `extern "C"` division, but never catches a divide-by-zero panic. When the host (say, a Node.js backend) passes a zero, Rust panics and tries to unwind the stack straight across the V8 boundary — instantly segfaulting the entire server process and every concurrent request it was serving, because of one unhandled failure in one native call.

**Example:** CPython's C API uses thread-local error state: a failing C extension returns `NULL` and calls `PyErr_SetString()` first; the interpreter checks the thread-local indicator on seeing `NULL` and raises the matching Python exception. SQLite, by contrast, relies entirely on return codes — nearly every call returns an `int` like `SQLITE_OK`, `SQLITE_BUSY`, or `SQLITE_ERROR`, with no hidden state anywhere. **[Strong Recommendation: default to return codes and out-parameters; reserve thread-local error state for integrating into a runtime that already mandates it]**

---

## Why Smart Engineers Disagree: "Thick" vs. "Thin" Bindings

The sharpest disagreement in FFI design is how much abstraction the binding layer in the host language should provide.

Engineers optimizing for native performance and mechanical transparency argue for thin bindings: the host-language layer should map 1:1 onto the C functions. If the C library needs three calls to initialize a struct, the Python developer makes those exact three calls with raw pointers. Wrapping it in Pythonic classes, to them, hides the true cost of every allocation and makes debugging the actual C mechanics nearly impossible.

Engineers optimizing for developer velocity and safety argue for thick bindings: forcing application developers to manage raw pointers and call `free()` by hand is a guaranteed source of leaks and segfaults. The FFI boundary should translate raw C structures into native, garbage-collected host objects immediately, with the C ABI treated as a hostile environment to be aggressively hidden behind idiomatic classes and automatic destructors.

The resolution is to build both, kept in strict isolation rather than conflated into one layer. First, a purely mechanical, unsafe, 1:1 thin binding (often literally named a `sys` module or crate) that mirrors the C ABI exactly, with no logic of its own. Then, a thick, idiomatic wrapper built on top of that thin layer, which is what application code actually consumes — fully insulated from the FFI mechanics, with an escape hatch back to the thin layer if a hot path ever needs to bypass the wrapper's overhead. Trying to make the FFI layer itself idiomatic produces a boundary that's simultaneously too slow for systems work and too unsafe for application work — the two goals only work as separate layers, never as one.

*Concepts expanded in later chapters: the broader practice of writing unsafe or low-level code (Part IV, Ch 33); concurrency and threading across a language boundary (Part X).*
