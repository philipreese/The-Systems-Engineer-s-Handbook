# Ch 74 — Shared State vs. Message Passing

**Prerequisites:** [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md) (throughput vs. reasoning simplicity), [Complexity Is the Enemy](../part01-systems-thinking/ch02-complexity-is-the-enemy.md) (state space explosion), [Cost Models and Mechanical Sympathy](../part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md) (MVCC vs. pessimistic locking), Principle 8, [FFI and Native Binding Design](../part03-api-design/ch26-ffi-and-native-binding-design.md), [When to Write Unsafe or Low-Level Code](../part04-code-organization/ch33-when-to-write-unsafe-or-low-level-code.md) (undefined behavior)

**New vocabulary introduced:** coordination model, shared-state concurrency, message-passing concurrency, data race

**Key takeaways:**
- A **coordination model** — shared state or message passing — is a decision made before and independent of any execution-unit choice (threads, async, processes; Ch 75). Conflating the two is the most common architectural mistake in concurrent design.
- [Strong Recommendation] Message passing eliminates the data race by construction, at the real cost of copying data and added latency per exchange. Shared state avoids that cost but requires continuous synchronization discipline — and discipline is exactly what fails silently under real production concurrency.
- A **data race** — two concurrent accesses to the same memory location, at least one a write, with no synchronization ordering them — is the concurrency-specific instance of the undefined behavior Ch 33 already defined, and is at its sharpest when the shared memory crosses an FFI boundary (Ch 26) with no runtime's safety guarantees active on either side.
- [Strong Recommendation] Rust's ownership and borrow-checker model (`Send`/`Sync`) is Principle 8 applied to concurrency specifically: catching a data race at compile time instead of relying on convention, documentation, or review to prevent it.
- Well-designed systems combine both models rather than picking one globally: shared mutable state stays small and local, message passing is exposed at every boundary that crosses a team, process, or service.

---

Part I named this Part's central tension by number: throughput versus reasoning simplicity (Ch 01). Concurrency is where that tension stops being abstract. Every later chapter in this Part assumes the choice this chapter makes has already been framed — Ch 75 goes deep on the primary tool for making shared state safe once it's chosen; Ch 76 separates a second, often-conflated axis (which execution unit actually runs the work) from the coordination question this chapter answers; Ch 77 catalogs how coordination goes wrong regardless of which model was chosen; Ch 78 goes deep on the actor model — Erlang/OTP's mailbox-and-isolation design is the extreme point of the message-passing spectrum — named here but not derived.

### Decision: Coordinate Concurrent Work Through Shared State or Message Passing

**What it is:** Whether concurrent units of execution operate on the same memory, with correctness enforced entirely by synchronization, or exclusively own private state and exchange copies of data through a channel, queue, or mailbox.

**Why it exists:** Sharing memory directly is the cheapest possible way to exchange information — no serialization, no copying, both sides observe the same object. That cheapness is exactly why it is dangerous: correctness now depends on every access, everywhere in the codebase, obeying synchronization rules, and that is a global reasoning burden that grows with the codebase rather than a local one. Message passing sidesteps the requirement for discipline instead of demanding it: if only one execution context ever owns a piece of state, nothing else can race with it, and the coordination problem becomes message ordering instead of memory correctness.

**Options:**
- **Shared state** — threads, processes, or async tasks access the same memory, protected by mutexes, read-write locks, or atomics (mechanics covered in Ch 75).
- **Message passing** — each execution context owns its state exclusively; communication happens by sending copies through channels, queues, mailboxes, or IPC.

**Trade-offs:**

| | Shared State | Message Passing |
|---|---|---|
| Throughput | High — zero-copy, direct access to large in-memory structures | Lower — every exchange copies data or transfers ownership |
| Correctness burden | Global — every accessor of the shared object must be considered together | Local — each unit reasons about its own private state alone |
| Failure signature | Silent, nondeterministic corruption, timing-dependent | Ownership and flow-control problems — recoverable, not silent |
| Natural fit | Caches, memory pools, shared indexes, kernels, database engines | Service orchestration, distributed workflows, team boundaries |

**When to choose each:** [Strong Recommendation] Shared state where copying the data would itself dominate execution cost and a latency budget is tight — database storage engines, kernels, memory allocators, high-frequency trading paths. Message passing as the default everywhere else — general business logic, service orchestration, anything where isolation, failure containment, or eventual distribution matters more than nanosecond-level throughput. The discipline shared state demands does not scale with team size or codebase age; message passing's cost is paid predictably, up front, per exchange.

**Common failure modes:** On the shared-state side: an engineer writes to a shared map from an async callback thread without acquiring a lock, and under production load the internal pointers corrupt into a crash or, worse, silently corrupted state served to users. On the message-passing side, different problems replace the data race entirely: a channel or mailbox grows without bound when producers outrun consumers — the general instance of Ch 08's Little's Law and backpressure argument, expanded in full for actor mailboxes specifically in Ch 78 — along with message-ordering assumptions and protocol mismatches between sender and receiver.

**Example:** POSIX threads expose shared memory directly to every thread; correctness depends entirely on programmers consistently protecting it with mutexes, and the OS provides the mechanism without guaranteeing correct use. Go embodies the opposite default through its own proverb — "Do not communicate by sharing memory; instead, share memory by communicating" — making channels a first-class, idiomatic way to move data between goroutines without exposing shared mutable state.

### Decision: Define the Data Race as Shared State's Governing Failure Mode

**What it is:** A data race occurs when two concurrent accesses target the same memory location, at least one is a write, and no synchronization establishes an ordering between them.

**Why it exists:** Compilers reorder instructions and modern CPUs reorder memory operations on the assumption that no concurrent observer sees an intermediate state unless synchronization says otherwise. Break that assumption and the language no longer defines what happens at all — a data race is not merely a wrong answer, it is the concurrency-specific instance of the undefined behavior Ch 33 already established.

**Options:** Prevent it through mutexes and locks (Ch 75 goes deep on this), atomic operations, ownership/borrow-checking systems, immutable data, or message passing, which removes the shared location entirely. Ignoring the possibility is not a viable option — an unsynchronized shared write is not a rare edge case, it is a certainty given enough concurrent traffic.

**Trade-offs:** Locks and atomics preserve shared-memory performance but must be used correctly at every single access site, forever. Ownership systems remove an entire bug class at compile time at the cost of real constraints on program structure. Message passing removes the race entirely at the cost of a communication tax paid on every exchange.

**When to choose each:** Synchronization is mandatory wherever multiple contexts must genuinely mutate the same object at once. Transferring ownership through a message is simpler wherever simultaneous ownership was never actually required — which is more often than shared-state code defaults to assuming.

**Common failure modes:** Crashes that disappear under a debugger because attaching one changes the timing that triggered the bug; corrupted internal state and invariant violations that "shouldn't be possible"; nondeterministic test failures reproducible only under real production load. The failure is sharpest at an FFI boundary (resolving Ch 26's deferred forward reference): a garbage-collected runtime's race detector has no visibility past the boundary it exposes a raw pointer across, so a Go program passing that pointer to a C library via cgo can pass its own race detector cleanly while a background C thread mutates the same memory concurrently — corrupting the process heap and crashing with no Go stack trace anywhere near the actual fault.

**Example:** A counter incremented simultaneously by multiple POSIX threads with no synchronization is the canonical instance — correct under light testing, wrong unpredictably once concurrency increases in production.

### Decision: Enforce Memory Safety Through the Compiler or Through Convention

**What it is:** Whether the guarantee that shared memory is accessed safely is checked mechanically, by the type system, at compile time, or left to programmer discipline — documentation, style guides, and code review.

**Why it exists:** Convention-based discipline is provably insufficient over any real codebase's lifespan. An unwritten rule ("only thread X may touch this map") is never visible to a compiler, and it is eventually violated by an engineer who joined after the rule was established and has no way to discover it exists. This is Principle 8 — mechanical enforcement over human discipline — applied to concurrency specifically.

**Options:** Convention-based discipline, where primitives (mutexes, atomics) are provided but correct use is entirely the programmer's responsibility and the compiler cannot detect a missing lock; or compile-time ownership and borrowing, where the type system rejects programs that would allow unsynchronized shared mutation before they can run at all.

**Trade-offs:** Convention gives maximum flexibility and the lowest initial cognitive cost, at the price of a codebase whose long-term safety depends on universal, permanent adherence to a rule that is frequently unwritten. Compiler-enforced ownership guarantees the absence of a data race in anything that compiles, with no runtime cost, at the price of a genuine upfront tax: code that would run fine in a less strict language is rejected until ownership is made explicit, slowing early prototyping.

**When to choose each:** [Strong Recommendation] Compile-time ownership for new systems-level infrastructure and any codebase expected to outlive its original authors. Convention-based discipline remains the realistic choice inside a mature, large C/C++ or Java codebase, where changing language or toolchain is not on the table regardless of the argument's merits.

**Common failure modes:** The lapsed convention — an implicit, undocumented rule that a new engineer never learns, violated the first time they call into the module from a thread the original author never anticipated, corrupting state silently with no compiler warning and no immediate crash pointing at the cause.

**Example:** Rust's borrow checker, together with its `Send` and `Sync` traits, rejects at compile time any attempt to share a non-thread-safe type across threads without an explicit synchronization wrapper — turning what would be an intermittent production failure in C into a build error. Go occupies a deliberate middle position: it makes channels first-class and idiomatic while still exposing `sync.Mutex`, nudging toward message passing as a default without removing shared-state concurrency as an option the way Rust's checker effectively does.

### Where Each Model's Boundary Belongs

Production systems rarely commit to one model exclusively. A database engine's storage layer runs fine-grained locks internally while exposing a purely message-based client protocol; a language runtime implements its channels using locks its callers never see. The open design question is not which model a system uses, but where each model's boundary sits. The pattern worth carrying into the rest of this Part: keep shared mutable state small and local, and expose message passing at every boundary that crosses a team, a process, or a service — exactly where synchronization discipline is hardest to enforce and hardest to verify after the fact.

### Why Smart Engineers Disagree

The disagreement is rarely about correctness — both sides agree a data race is a bug. It is about which cost is worse to pay: the performance tax of copying data, or the reasoning tax of shared-memory discipline.

Engineers building database storage engines, kernels, memory allocators, or high-frequency trading systems tend to view message passing as wasteful by default: copying large structures across channels triggers cache-line evictions and heap allocations that a direct shared-memory read never pays, and at their latency budgets that cost is not negligible. Their answer is mechanical discipline — carefully mapped critical sections, fine-grained locks, or lock-free atomics — because the alternative genuinely costs more in the domain they operate in.

Engineers building distributed services and business systems tend to view manual synchronization discipline as the anti-pattern: it fails deterministically under deadline pressure and team turnover, and the cost of debugging one production data race — hours or days of tracing a nondeterministic failure — outweighs the hardware cost of copying a message. They accept the throughput tax to make memory boundaries absolute.

Neither position is wrong; each is optimizing for a different target this handbook has already named — raw hardware throughput versus reasoning simplicity and engineering velocity (Ch 01). The mistake is assuming one model universally dominates the other rather than asking which cost the specific boundary in front of you can actually afford.
