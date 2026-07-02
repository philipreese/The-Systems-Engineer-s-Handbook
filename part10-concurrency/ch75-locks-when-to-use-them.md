# Ch 75 — Locks: When to Use Them

**Prerequisites:** [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md) (Principle 6), [Cost Models and Mechanical Sympathy](../part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md) (MVCC vs. pessimistic locking), [Shared State vs. Message Passing](ch74-shared-state-vs-message-passing.md) (data race, shared-state concurrency)

**New vocabulary introduced:** critical section, lock granularity, lock contention, optimistic concurrency control (OCC)

**Key takeaways:**
- A lock protects an invariant across a **critical section**, not just a single variable — every read and write that could observe or break that invariant must happen while the lock is held, or the lock provides no guarantee at all.
- [Strong Recommendation] Default to coarse-grained locking — one lock protecting a larger region of state. It is easier to reason about and harder to get wrong. Move to fine-grained locking only after profiling (Part XII, Ch 86) has actually shown lock contention is the real bottleneck, not before.
- Fine-grained locking buys real throughput under contention, at a real cost: exponentially more possible interleavings to reason about and meaningfully higher deadlock risk once multiple locks must be acquired together (cataloged in Ch 77, not derived here).
- Read-write locks, optimistic concurrency control, and MVCC (already established in Ch 06) are legitimate alternatives to a plain mutex, each suited to a specific contention shape rather than being a universal upgrade.
- No mature runtime uses one synchronization primitive everywhere — the Linux kernel's mutex/spinlock/RCU split exists because the correct tool depends on the workload's latency and contention profile, not on sophistication for its own sake.

---

Ch 74 established that shared state requires synchronization to stay correct; this chapter goes deep on the primary tool for providing it. A lock guarantees that only one execution context enters a protected critical section at a time, converting a data race (Ch 74) into an explicit, deliberate ordering of accesses. The question this chapter answers is not whether to use a lock — Ch 74 already settled that shared state needs protecting — but how much state a single lock should protect, and when a lock is the wrong tool entirely.

### Decision: Set Lock Granularity Coarse First, Fine Only After Profiling

**What it is:** How much shared state a single lock protects — from one lock guarding an entire subsystem (coarse-grained) to many independent locks each guarding a narrow slice of it, down to per-object or per-bucket granularity (fine-grained).

**Why it exists:** The primary cost of concurrent software is rarely the lock's own overhead — it is the reasoning complexity that comes from many independently synchronized regions. Every additional lock multiplies the number of possible execution interleavings a reader must hold in their head to verify correctness. A single lock minimizes that combinatorial cost; splitting it multiplies both available parallelism and the number of ways the system can go wrong.

**Options:**
- **Coarse-grained locking** — one lock protecting an entire data structure, subsystem, or object tree.
- **Fine-grained locking** — the same state partitioned into independent regions (per bucket, per object, per shard), each with its own lock, allowing unrelated operations to proceed truly concurrently.

**Trade-offs:** Coarse-grained locking gives a straightforward correctness model, a low review burden, and zero risk of multi-lock interaction hazards — at the cost of serializing operations that don't actually conflict, capping throughput under real contention. Fine-grained locking raises throughput and CPU utilization under contention, at the cost of substantially more synchronization logic to get right and a meaningfully higher risk of deadlock once an operation must hold more than one of the finer locks at once (Ch 77).

**When to choose each:** [Strong Recommendation] Coarse-grained locking as the default for any new system, any system where contention hasn't been measured, or where correctness is still evolving — this is Principle 6 applied directly: don't spend reasoning complexity before evidence justifies the expense. Move to finer granularity only once profiling (Part XII, Ch 86 covers the methodology; it isn't derived here) demonstrates that lock contention, not computation, is actually limiting throughput, and the data can be partitioned along a boundary with no overlapping transactional requirements.

**Common failure modes:** Premature fragmentation — an engineer splits a cache into dozens of bucket-level locks to satisfy a contention problem that was never measured, and an operation touching two buckets now has to acquire multiple locks, introducing a silent deadlock risk that static testing never exercises and that only surfaces under a real, non-linear production spike.

**Example:** Java's collection classes trace this exact graduation in public API history. `Vector` and `Hashtable` used one coarse, exclusive lock per method call, which serialized every access and stalled multi-threaded servers under load. `ConcurrentHashMap` replaced that single table-wide lock with an array of independent bucket-level locks, allowing genuinely parallel writes to different regions — at the cost of measurably more complex internal implementation.

### Decision: Reach for a Read-Write Lock Only When Reads Dominate

**What it is:** Whether a critical section requires strict, binary exclusivity for every accessor, or can distinguish readers — who may proceed concurrently with each other — from writers, who still require exclusive access.

**Why it exists:** Many workloads read far more often than they write. Serializing every reader behind an ordinary mutex wastes parallelism that a read-write lock can recover, since concurrent reads don't threaten the invariant a plain mutex was protecting against writers in the first place.

**Options:** An ordinary exclusive mutex, or a read-write lock (`RWMutex`) that allows unlimited concurrent readers but still serializes writers against everyone.

**Trade-offs:** An ordinary mutex has lower bookkeeping overhead and faster acquisition in a balanced or write-heavy workload. A read-write lock recovers substantial reader-side throughput in a read-dominated workload, but carries heavier internal state-tracking overhead than a plain mutex, and a poorly chosen fairness policy can let a continuous stream of new readers starve a pending writer indefinitely.

**When to choose each:** A read-write lock only where reads outnumber writes by a wide margin and the read-side critical section is long enough to make the extra bookkeeping worth it — a routing table or a service-discovery map updated rarely and read on every request. An ordinary mutex otherwise; it is simpler, and for a balanced or write-heavy workload it is often faster too.

**Common failure modes:** Writer starvation — a team migrates a hot cache lock from a plain mutex to a read-write lock expecting a pure win. Under a sustained flood of reads, new readers keep joining the shared lock ahead of a pending writer, and a background thread trying to flush updated configuration is blocked indefinitely while the served data quietly goes stale.

**Example:** Go exposes both `sync.Mutex` and `sync.RWMutex` as peers, not a basic-to-advanced progression — engineers reach for `RWMutex` specifically for something like an in-memory routing table read by thousands of concurrent request-handling goroutines and written by a single background updater once a day.

### Decision: Weigh Pessimistic Locking Against Lock-Avoiding Alternatives

**What it is:** Whether to physically block a thread out of a critical section until it can acquire exclusive access (pessimistic locking), or let operations proceed unblocked and only detect and resolve conflicts after the fact (lock avoidance).

**Why it exists:** Blocking a thread on an OS-level mutex is expensive — it triggers a context switch and a kernel transition. When conflicts between concurrent operations are genuinely rare, paying that cost on every single access wastes it on the common case where no conflict was ever going to occur.

**Options:**
- **Pessimistic locking** — a thread blocks until it acquires the lock it needs.
- **Optimistic concurrency control (OCC)** — a thread executes speculatively against a local copy, then checks at commit time (typically via a hardware compare-and-swap) whether the underlying data changed underneath it, retrying the whole operation if it did.
- **MVCC** — already established in Ch 06 as PostgreSQL's specific instance of avoiding read-write contention through versioning rather than locking; referenced here as one point on this same spectrum, not re-derived.
- **Lock-free algorithms** — data structures built directly on atomic hardware primitives with no blocking at all.

**Trade-offs:** Pessimistic locking gives a flat, predictable latency ceiling even under heavy, sustained contention, at the cost of unnecessary kernel scheduling overhead when conflicts are actually rare. OCC gives near-native performance and zero blocking when conflicts are rare, but performance collapses under high write contention, when threads spend their time retrying instead of making progress. MVCC avoids read-write blocking entirely by giving readers a consistent snapshot while writers create new row versions, at the storage and vacuuming cost Ch 06 already detailed. Lock-free algorithms avoid blocking altogether but are exceptionally difficult to design correctly, depend on subtle memory-ordering semantics, and are far harder to review and maintain than an equivalent lock — they solve a narrower problem than conventional locking, not a strictly better one.

**When to choose each:** [Strong Recommendation] Pessimistic locking when write contention is genuinely high or the system runs under a latency ceiling where a retry storm is unacceptable. OCC when conflicts are statistically rare and most attempts will succeed on the first try. MVCC where the storage model already fits a versioning approach — reference Ch 06's PostgreSQL trade-off directly rather than re-litigating it. Lock-free algorithms only for a component whose bottleneck has actually been measured (Part XII, Ch 86) and whose correctness can be rigorously validated — a core scheduler, a low-level allocator, a specialized high-throughput ring buffer — not as a default replacement for a working lock.

**Common failure modes:** A retry storm under OCC — thousands of concurrent workers attempt to increment the exact same counter during a traffic spike; exactly one succeeds per round, the rest loop and retry, and CPU utilization saturates while end-to-end latency climbs exponentially, which is optimistic concurrency's version of livelock (Ch 77 covers livelock as a named failure mode in full).

**Example:** The Linux kernel doesn't pick one strategy — it matches the tool to the workload. Mutexes protect longer critical sections where sleeping is acceptable; spinlocks protect very short sections where the cost of sleeping would exceed the cost of a brief busy-wait; and RCU (Read-Copy-Update) lets readers traverse read-mostly structures with no lock and no atomic instruction at all, while writers build a new copy and swap a pointer atomically — the same versioning idea MVCC applies at the database layer, applied here in kernel memory.

The lesson isn't that every system needs all three techniques. It's that the correct synchronization mechanism is chosen against a workload's actual latency and contention profile — the same evidence-before-optimization discipline Principle 6 already states, applied here specifically to locking strategy rather than treated as a separate rule.

### Why Smart Engineers Disagree on Lock-Free Code

The disagreement here isn't about locking granularity — the profiling-first, coarse-by-default position is not seriously contested. It's specifically about whether to replace a working lock with a lock-free data structure built on raw atomic primitives.

One position treats a traditional lock as an admission of defeat: blocking destroys real-time latency guarantees, introduces jitter that's hard to bound, and scales poorly as core counts climb. This position advocates building critical structures directly out of atomic compare-and-swap operations so the system always makes forward progress without ever asking the kernel to schedule anything.

The opposing position treats lock-free code with real skepticism: it isn't merely hard, it is a discipline where a single out-of-order memory access produces silent corruption that can take months to trace back to its cause, and verifying correctness requires reasoning about memory-visibility orderings most engineers never need to think about otherwise. To this position, a coarse, well-placed mutex is the better engineering choice precisely because its safety boundary is explicit and legible to the next engineer who reads it.

The resolution isn't a coin flip between styles — it's a question of how rare the operation actually is and how catastrophic the failure mode of getting it wrong would be. Outside a core scheduler, a low-level allocator, or a specialized high-throughput ring buffer, the performance gain a lock-free structure offers rarely justifies the risk of an untraceable memory-ordering bug; a well-scoped, coarse-grained lock remains the correct default for the overwhelming majority of systems.
