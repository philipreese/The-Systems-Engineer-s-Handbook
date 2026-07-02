# Part X — Chapter Specifications

Pre-filled special instructions for each Part X chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

Prerequisites for Part X: Part I (Ch 01's optimization-target framing sets up this Part's central
tension by name — "locks vs. message passing" as a throughput-vs-reasoning-simplicity trade-off,
almost verbatim the frame Ch 74 and Ch 75 must deliver on; Ch 02's state space explosion is the
precise, already-defined reason concurrent bugs are exceptionally hard to reproduce and test,
central to Ch 77; Ch 06 already established MVCC vs. pessimistic locking as a database-specific
instance of avoiding lock contention, which Ch 75 generalizes rather than re-derives; Ch 07's
fail-fast principle and MTTR-over-MTBF framing is what Ch 78's supervision-tree pattern applies at
process-isolation granularity instead of whole-system granularity; Ch 08's Little's Law and
backpressure are the direct mechanism behind Ch 78's mailbox-overflow failure mode); Part III
(Ch 26's FFI chapter deferred "concurrency and threading concerns across a language boundary" to
this Part in general terms); Part IV (Ch 33's unsafe/low-level code chapter deferred
"concurrency-specific unsafe patterns" to this Part in general terms — both resolved here, in
Ch 74, where an unsynchronized data race is the concurrency-specific instance of Ch 33's
already-defined undefined behavior, at its sharpest when the shared memory in question crosses an
FFI boundary with no host-runtime guard on either side).

Boundary with adjacent Parts: Part XII (Performance) lists this Part as an explicit prerequisite —
Part X decides which concurrency model and coordination strategy is correct for a given problem;
Part XII decides how to measure and optimize performance once a model is already chosen. Any
chapter here that starts discussing benchmarking methodology, profiling technique, or algorithmic
complexity should stop and flag it to Part XII (Ch 85, Ch 86, Ch 89), not become a performance-
tuning guide itself. Part I, Ch 06 owns the database-specific shared-state-vs-versioning trade-off
(MVCC vs. pessimistic locking); this Part's Ch 75 generalizes locking as a mechanism beyond that
one example, referencing it rather than re-arguing it. Part V owns general testing methodology;
Ch 77 only names why concurrent bugs are exceptionally hard to catch, it doesn't reopen test
strategy as a topic.

This Part is structured as one foundational tension, two axes that are routinely conflated with
it, one catalog of failure modes, and one paradigm that sidesteps the whole category: Ch 74
decides the fundamental coordination model — shared state or message passing — and is this Part's
spine. Ch 75 goes deep on the primary tool for making shared state safe once it's chosen. Ch 76
separates a second, often-conflated axis — which execution unit actually runs the work (async,
threads, or processes) — from the coordination-model question Ch 74 already answered; a reader
should finish it able to explain why "async vs. threads" and "locks vs. message passing" are two
different questions, not one. Ch 77 catalogs precisely how coordination goes wrong regardless of
which model was chosen. Ch 78 closes the Part with the one paradigm — the actor model — that
eliminates Ch 77's entire failure category by construction rather than by discipline, at the cost
of new trade-offs of its own.

---

## Ch 74 — Shared State vs. Message Passing

```
This chapter opens Part X by delivering the tension Ch 01 already named by number: locks vs.
message passing as a throughput-vs-reasoning-simplicity trade-off. Every later chapter in this
Part assumes this choice has already been framed, so treat it as the Part's spine, not one option
among many.

Cover the two fundamental coordination models precisely. Shared state: multiple concurrent units
of execution read and write the same memory, and correctness depends entirely on synchronizing
access to it — the mechanism Ch 75 goes deep on. Message passing: concurrent units of execution
own their own state exclusively and communicate only by sending copies of data through a channel,
so there is no shared memory to protect in the first place. Take a position: message passing
eliminates an entire category of bug (the data race) by construction, at the real cost of copying
data and added latency per exchange; shared state avoids that copying cost but requires ongoing
discipline — or compiler-enforced guarantees — to stay correct, and that discipline is exactly
what fails silently under real production concurrency.

Cover the data race as the central, precisely defined failure mode of the shared-state side: two
concurrent accesses to the same memory location, at least one a write, with no synchronization
between them. Name this explicitly as the concurrency-specific instance of the undefined behavior
Ch 33 already defined — this is where Ch 33's deferred "concurrency-specific unsafe patterns"
forward reference resolves. Resolve Ch 26's deferred FFI forward reference here too: a data race is
at its sharpest, highest-stakes point when the shared memory in question crosses a language
boundary with no host runtime's safety guarantees active on either side.

Anchor in real systems: Go's "share memory by communicating" proverb and its channel primitive as
the language-level embodiment of the message-passing side; POSIX threads and mutexes as the
traditional shared-memory-with-manual-synchronization baseline; Rust's ownership and borrow-checker
model as a concrete instance of Principle 8 — data races caught at compile time instead of enforced
by convention or code review. Place the actor model (Erlang/OTP) as the extreme point on the
message-passing end of the spectrum, named but not derived — Ch 78 owns it in full.

Do NOT cover: lock implementation, granularity, or acquisition-ordering strategy in depth (Ch 75 —
this chapter only establishes that shared state needs protecting, not how); the mechanics of
threads, async, or OS processes as execution units (Ch 76 — a distinct axis from the coordination
model this chapter covers); the actor model's supervision and fault-tolerance mechanics (Ch 78,
referenced only as the far end of the message-passing spectrum); deadlock, livelock, and starvation
as named failure modes (Ch 77).
```

---

## Ch 75 — Locks: When to Use Them

```
This chapter goes deep on the primary tool for making Ch 74's shared-state option safe: the lock.
It assumes Ch 74 already established why shared state needs protecting at all: this chapter is
about how, and when a lock is the wrong tool entirely.

Cover lock granularity as the chapter's central trade-off, and take a position consistent with this
handbook's premature-optimization throughline (Principle 6): default to coarse-grained locking — a
single lock protecting a larger region — because it's easier to reason about and harder to get
wrong, and move to fine-grained locking (many smaller locks, each protecting less) only once
profiling (referenced forward to Part XII, Ch 86 — not derived here) has actually shown lock
contention is the real bottleneck. Name the cost of fine-grained locking honestly: better
throughput under contention, at the cost of exponentially more interleavings to reason about and
meaningfully higher deadlock risk (forward-referenced to Ch 77 by name, not derived here).

Cover lock-free and lock-avoiding alternatives as real options, not strawmen: optimistic
concurrency control (attempt the operation, detect conflict after the fact, retry), read-write
locks for read-heavy workloads where most access doesn't need exclusivity, and Ch 06's
already-established MVCC as the database-specific instance of avoiding locking entirely through
versioning — reference Ch 06's PostgreSQL example directly rather than re-deriving the trade-off it
already made.

Anchor in real systems: the Linux kernel's graduated lock strategies (mutex, spinlock, RCU) matched
to different contention and latency profiles as a real system that doesn't use one lock type
everywhere; Go's `sync.Mutex` and `sync.RWMutex`, or Java's `synchronized` keyword and
`ReentrantLock`, as concrete language-level primitives implementing the coarse-vs-fine and
exclusive-vs-shared distinctions this chapter covers.

Do NOT cover: what happens when multiple locks are acquired out of consistent order across threads
(Ch 77 — this chapter covers correct use of a lock in isolation, not the multi-lock interaction
failure); message passing as the alternative to locking altogether (Ch 74, already covered); the
choice of execution unit — thread, process, or async task — holding the lock (Ch 76); general
database query optimization or index design (Part XII).
```

---

## Ch 76 — Async vs. Threads vs. Processes

```
This chapter separates a second axis from the one Ch 74 and Ch 75 already covered: not how
concurrent units coordinate access to shared data, but which execution unit actually runs
concurrent work in the first place. A reader confusing "should I use a lock or a channel" with
"should I use a thread or async" is conflating two different decisions; resolving that conflation
explicitly is this chapter's job before anything else.

Cover the concurrency-vs-parallelism distinction precisely and early, since it's the most routinely
conflated pair of terms in this Part: concurrency is structuring a program to handle more than one
unit of work in overlapping time (doesn't require multiple cores); parallelism is literally
executing more than one unit of work at the same instant (requires multiple cores). Cover the three
execution models against that distinction: OS processes (strongest isolation, separate memory
space, highest overhead to create and context-switch, true parallelism across cores); OS threads
(share memory within a process, lighter than processes, still preemptively scheduled by the OS,
true parallelism across cores); async/coroutines (cooperative scheduling, typically single-threaded
or few-threaded, non-blocking I/O, lowest per-unit overhead, no CPU parallelism without an explicit
thread pool underneath it).

Cover the actual selection test as the chapter's central deliverable: CPU-bound work needs true
parallelism (threads or processes on multiple cores) to go faster; I/O-bound work needs concurrency
without needing extra cores at all (async is often strictly cheaper). Name the central failure mode
directly: using threads for I/O-bound work pays context-switch and memory overhead that bought
nothing; using async for CPU-bound work blocks the single event loop and stalls every other
concurrent task sharing it, which is a worse failure than the one async was chosen to avoid.

Anchor in real systems: Python's Global Interpreter Lock (GIL) as a concrete, widely known example
of a runtime where OS threads don't buy CPU parallelism at all; Node.js's single-threaded event
loop as the canonical async-first runtime, built specifically for I/O-bound workloads; Go's
goroutines as a deliberate hybrid — lightweight, and M:N scheduled onto a small pool of OS threads
by the Go runtime itself, blurring the thread/async line by design.

Do NOT cover: shared-state coordination or locking mechanics once an execution model is chosen
(Ch 74, Ch 75 — this chapter is about which unit runs the work, not how units sharing state
coordinate); the actor model as a higher-level concurrency abstraction (Ch 78); profiling or
benchmarking methodology for measuring which model performs better on a given workload (Part XII,
Ch 86 — this chapter covers the structural trade-off, not the measurement process).
```

---

## Ch 77 — Deadlock, Livelock, and Starvation

```
This chapter catalogs precisely how coordination goes wrong, building directly on Ch 75 (locks are
the most common cause) and Ch 76 (any of the three execution models can starve or deadlock). It is
the direct, concrete instance of Ch 02's state space explosion: the combinatorial number of
possible interleavings between concurrent operations is exactly why these failures are
non-deterministic, timing-dependent, and notoriously hard to reproduce — reference Ch 02's term
directly rather than reinventing it.

Cover the three failure modes as precisely distinct, since they're routinely conflated in casual
engineering conversation: deadlock (a set of units are each waiting on a resource held by another
in the set, forming a cycle — total, permanent standstill); livelock (units are actively responding
to each other and changing state, but no actual progress is made — looks like activity, isn't);
starvation (a unit is correct and able to proceed, but a scheduling or lock-acquisition policy
perpetually deprioritizes it in favor of others).

Cover the four Coffman conditions for deadlock precisely (mutual exclusion, hold-and-wait, no
preemption, circular wait), and take a position: breaking any single one of the four is sufficient
to prevent deadlock structurally, and enforcing a single, global, consistent lock-acquisition order
across a codebase — eliminating circular wait specifically — is the cheapest and most common
practical technique, because it doesn't require giving up mutual exclusion, hold-and-wait
tolerance, or preemption capability to work.

Anchor in real systems: the dining philosophers problem as the canonical, widely taught illustration
of deadlock via circular wait; a real database engine's deadlock detector (PostgreSQL or MySQL
InnoDB automatically detecting a wait-for cycle among transactions and killing one as the "victim"
to break it) as a concrete example of runtime detection and recovery, distinct from pure prevention
via lock ordering.

Do NOT cover: general concurrency testing strategy, stress-testing tooling, or fuzzing methodology
in depth (name the state-space-explosion reason testing is hard; do not expand into a testing
methodology chapter — no Part in this handbook owns concurrent-specific test strategy as a
dedicated topic, so keep this to the failure-mode explanation only); the actor model's structural
avoidance of shared-memory deadlock entirely (Ch 78 — referenced as one way to sidestep the whole
category, not re-derived here).
```

---

## Ch 78 — The Actor Model

```
This chapter closes Part X with the one paradigm that eliminates Ch 77's entire failure category by
construction rather than by discipline. It resolves the message-passing end of Ch 74's spectrum in
full depth, the same way Ch 72 went deep on the one signal Ch 70 had only placed within a taxonomy.

Cover the actor model's core properties precisely: an actor is an isolated unit of computation with
private state no other actor can directly access or mutate; actors communicate exclusively through
asynchronous messages delivered to a mailbox; each actor processes exactly one message at a time
from its own mailbox. The direct consequence: there is no shared memory between actors, and
therefore no data race is possible between them — a structural elimination of Ch 74 and Ch 75's
entire failure category, not a discipline-based mitigation of it. Name this explicitly as a
concrete instance of Principle 8 (mechanical enforcement over human discipline), applied here at
the concurrency-architecture level instead of the type-system level.

Cover supervision trees and "let it crash" as the actor model's structural answer to fault
tolerance: an actor that hits an unexpected error is allowed to crash outright and be restarted by
a supervising actor into a known-good state, rather than defensively coded to anticipate and
survive every conceivable failure inline. Connect this directly and explicitly to Ch 07's
already-established fail-fast principle and MTTR-over-MTBF framing — the same argument, applied at
process-isolation granularity instead of whole-system granularity.

Cover the trade-off honestly, not just the benefit: actors introduce their own new failure mode — an
actor's mailbox can grow unboundedly if messages arrive faster than the actor processes them, which
is a direct, concrete instance of Ch 08's already-established backpressure and Little's Law
argument (reference it directly, don't re-derive it) — and actor-to-actor messaging carries real
serialization and latency cost that direct shared-memory access inside a single process never pays.

Anchor in real systems: Erlang/OTP and the BEAM VM as the actor model's origin and most mature
real-world proof point, built for telecom-grade fault tolerance; Akka on the JVM as a widely used
adoption of the same model outside its native ecosystem.

Do NOT cover: message passing as a general alternative to shared state (Ch 74 already places actors
on that spectrum at a high level; this chapter goes deep on the actor model specifically, not on
plain channels or message queues as a general pattern); distributed consensus or coordination
between actors running on physically separate machines (genuinely out of scope for this handbook —
flag it as a real, unresolved boundary rather than deferring to a chapter that doesn't own it).
```
