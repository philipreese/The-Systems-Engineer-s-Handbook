# Ch 77 — Deadlock, Livelock, and Starvation

**Prerequisites:** [Complexity Is the Enemy](../part01-systems-thinking/ch02-complexity-is-the-enemy.md) (state space explosion), [Shared State vs. Message Passing](ch74-shared-state-vs-message-passing.md), [Locks: When to Use Them](ch75-locks-when-to-use-them.md) (lock granularity), [Async vs. Threads vs. Processes](ch76-async-vs-threads-vs-processes.md)

**New vocabulary introduced:** deadlock, livelock, starvation, Coffman conditions

**Key takeaways:**
- Deadlock, livelock, and starvation are precisely distinct, not interchangeable words for "concurrency is broken": deadlock is a permanent, total standstill; livelock is continuous activity with no forward progress; starvation is one correct participant perpetually denied its turn while everyone else proceeds normally.
- [Strong Recommendation] All four Coffman conditions — mutual exclusion, hold-and-wait, no preemption, circular wait — must hold simultaneously for deadlock to occur. Breaking any single one prevents it, and enforcing a single, consistent global lock-acquisition order is the cheapest, most common technique, because it only requires giving up circular wait, not the other three.
- Prevention and detection-and-recovery are different strategies for different contexts: structural lock ordering fits a codebase the engineering team fully controls; runtime cycle detection (a wait-for graph, a chosen "victim" transaction) fits a system like a database where access patterns are dictated by arbitrary, dynamically generated queries that can't be pre-ordered.
- Livelock is broken by randomizing retry timing, not by retrying faster — symmetric, deterministic backoff is exactly what synchronizes participants into a lockstep collision loop in the first place.
- These failures are the direct, concrete instance of Ch 02's state space explosion: the combinatorial number of possible interleavings between concurrent operations is why they are non-deterministic, timing-dependent, and can vanish the moment a debugger or extra logging changes the timing that exposed them.

---

Ch 75 established locks as the primary tool for making shared state safe, and Ch 76 separated the execution unit from that coordination question. This chapter asks what happens when the coordination those chapters covered goes wrong — not from incorrect logic, but from the sheer number of ways independently executing operations can interleave. This is Ch 02's state space explosion made concrete: every additional concurrent execution path multiplies the number of possible orderings, and only a vanishingly small fraction of them need to be wrong for a failure to eventually occur in production, however cleanly the same code ran the previous million times. That same combinatorics is why this chapter stops at explaining the failure modes and does not expand into a concurrent-testing methodology — no part of this handbook owns that as a dedicated topic, and exhaustively enumerating interleavings is not a tractable testing strategy to teach.

Three distinct failures share this root cause. **Deadlock** is a set of execution units each waiting indefinitely for a resource held by another member of the same set — a cycle, with no participant able to proceed and none willing to release what it already holds. **Livelock** looks like the opposite: units remain active, continuously changing state and responding to each other, but no useful work ever completes. **Starvation** is narrower still: one correct, ready-to-proceed unit is perpetually denied its turn by a scheduling or lock-acquisition policy that favors others, while the rest of the system continues operating normally. All three are failures of progress, not of logic — the code involved may be entirely correct in isolation.

### Decision: Prevent Deadlock Through Lock Ordering, or Detect and Recover From It

**What it is:** Whether to eliminate deadlock structurally, by enforcing a single global order in which locks are acquired, or to permit flexible, dynamic lock acquisition and rely on a runtime mechanism to detect a cycle and break it after the fact.

**Why it exists:** Deadlock requires all four Coffman conditions to hold at once: mutual exclusion (a resource can't be shared), hold-and-wait (a unit holds one resource while requesting another), no preemption (a resource can't be forcibly taken back), and circular wait (a cycle of units each waiting on the next). Breaking any single condition is sufficient to prevent deadlock, but mutual exclusion, hold-and-wait, and no-preemption are usually load-bearing properties a system needs for other reasons. Circular wait is the one condition that can be eliminated for free: if every execution path acquires locks in the same global order, a cycle can never form, because no path can legally acquire a lower-ordered lock after a higher-ordered one.

**Options:**
- **Global lock ordering** — every lock in the system is assigned a consistent acquisition order; any path that might hold two locks at once must acquire them in that order, with no exceptions.
- **Runtime cycle detection and recovery** — locks are acquired dynamically in whatever order the workload demands, while a background mechanism tracks a wait-for graph of which unit is waiting on which resource and breaks any cycle it finds.

**Trade-offs:** Global lock ordering costs nothing at runtime and eliminates the most common cause of deadlock entirely, at the price of an ongoing, codebase-wide discipline: every new lock has to be placed correctly in the hierarchy, and a single acquisition made out of order silently breaks the guarantee for everyone. Runtime detection gives up that upfront discipline requirement — engineers can acquire resources in whatever order a dynamic workload actually needs — but requires a real tracking mechanism, degrades throughput under heavy contention as the cycle check runs, and forces something to happen when a cycle is found: a chosen transaction gets aborted and rolled back, and the caller has to handle that outcome as a normal part of operation.

**When to choose each:** [Strong Recommendation] Global lock ordering as the default for a codebase the engineering team fully controls — kernels, application frameworks, in-process data structures — because it prevents the defect before a single line executes. Runtime detection specifically where lock-acquisition order can't be predicted at all, because it's dictated by arbitrary, dynamically generated requests — a relational database executing whatever transactions its callers send it is the clearest case, not an exception to prevention but a context where prevention isn't available.

**Common failure modes:** An undocumented convention — "always lock the metadata table before the inventory table" — holds until an emergency hotfix, written under incident pressure, updates inventory and then logs metadata, acquiring the two locks in the opposite order. The violation sits latent until the next real traffic surge interleaves the hotfix's code path with the original one, and the system deadlocks permanently, forcing a manual restart.

**Example:** The dining philosophers problem is the canonical illustration: five philosophers each need both adjacent forks to eat, and if every one picks up their left fork at once, all five wait forever for a fork their neighbor already holds — a pure circular wait, resolved entirely by a rule as simple as "always pick up the lower-numbered fork first." PostgreSQL and MySQL's InnoDB take the opposite, detection-based approach for exactly the reason above: transactions lock rows in whatever order their SQL demands, the engine watches the resulting wait-for graph, and when a cycle appears it aborts one transaction as the "victim," releases its locks, and lets the rest proceed — accepting that deadlocks will occur in exchange for never requiring a global lock order across arbitrary user queries.

### Decision: Break Livelock by Randomizing Retries, Not Retrying Faster

**What it is:** Whether concurrent units that fail to acquire a resource retry immediately, on a fixed schedule, or after a randomized delay before trying again.

**Why it exists:** Livelock is what well-intentioned conflict-avoidance logic produces by accident: if every participant responds to a collision the same deterministic way — back off immediately, retry immediately — then participants with synchronized timing collide, back off, and retry in lockstep forever. The system is doing real work in the sense that CPU cycles are spent and state is changing; none of it is useful, because every participant makes the identical decision every time.

**Options:** Deterministic, fixed-interval or immediate retry the instant a conflict is detected; or randomized backoff — often exponential — that adds jitter so competing participants no longer share the same timing.

**Trade-offs:** Deterministic retry minimizes latency when contention is genuinely rare, since it grabs a freed resource the instant it's available, with no added complexity. But under synchronized contention it can collapse into perfect lockstep collision, burning all available CPU on retries that never succeed. Randomized backoff breaks that symmetry and guarantees eventual progress even under sustained contention, at the cost of real, intentionally-introduced latency jitter on individual attempts.

**When to choose each:** Deterministic retry only inside tightly serialized, single-threaded or cooperative contexts where a true multi-party symmetric collision structurally can't occur. Randomized backoff as the default everywhere real contention between independent participants is possible — optimistic-concurrency retries (Ch 75), distributed lock acquisition, and network-level retries alike.

**Common failure modes:** A lock-free ring buffer has several worker threads contend for the same head pointer; when a compare-and-swap fails, each thread immediately retries. Under load, several threads fall into an identical cadence, colliding and retrying in perfect synchrony — CPU usage pins at 100% while actual throughput drops to zero.

**Example:** Ethernet's CSMA/CD solved exactly this problem at the physical layer decades before it became a software concern: when two network interfaces collide transmitting on a shared wire, each picks a randomized delay drawn from a range that doubles with each successive collision (truncated binary exponential backoff), so the two nodes' timing separates instead of staying locked together, and one of them claims the wire.

### Decision: Choose Fair or Unfair Lock-Acquisition Scheduling

**What it is:** Whether a lock hands ownership strictly to whichever waiter arrived first (fair, FIFO), or lets a thread that's already running on a core seize a lock the instant it's released, ahead of threads that have been waiting longer (unfair, barge-in).

**Why it exists:** Starvation is what an acquisition policy produces when it's under no obligation to be fair: a scheduler or lock implementation that always favors the requester already running wins on raw throughput, because waking a sleeping thread costs real time — a real instance of Ch 06's mechanical sympathy, since the CPU core sits idle during that wake-up window regardless. But nothing in that policy guarantees the waiting thread ever gets its turn.

**Options:** Fair (FIFO) scheduling, which queues waiters and always hands the lock to the oldest one; or unfair (barge-in) scheduling, which lets an already-running thread grab a freshly released lock ahead of the queue.

**Trade-offs:** Fair scheduling guarantees every waiter eventual progress and bounds tail latency, at the cost of idling a CPU core during every wake-up handoff and lower peak throughput overall. Unfair scheduling maximizes throughput by keeping hot threads running and avoiding that handoff cost, at the cost of a real risk that a cold, sleeping thread gets bypassed indefinitely under sustained contention.

**When to choose each:** Unfair scheduling by default for general-purpose, throughput-oriented locks, which is most of them. Fair scheduling, or a hybrid, specifically where a bounded worst-case latency matters more than average throughput, or where a specific waiter holds an invariant the rest of the system needs flushed.

**Common failure modes:** A custom read-write lock lets new readers keep joining as long as the lock is held in shared mode. Under a sustained flood of read traffic, a background writer waiting to update configuration is never admitted — the read-side traffic never actually stops long enough to hand it the lock — and the service keeps serving stale data until it's restarted.

**Example:** Go's `sync.Mutex` runs unfair by default for throughput, but tracks how long the longest waiter has been blocked; once that exceeds roughly one millisecond, the mutex switches into an explicit "starvation mode" where new arrivals are barred from barging in and ownership passes directly down the wait queue, reverting to unfair mode once the queue clears. It treats fairness as a graduated safety valve applied only when starvation has actually become measurable, not as a permanent policy chosen up front.

### Why Smart Engineers Disagree on Static Enforcement vs. Runtime Mitigation

The disagreement generalizes past deadlock specifically to a question about where correctness should be guaranteed: at compile time, or at runtime.

One position treats any code that can compile with a latent circular-wait possibility as a structural failure. It favors languages and tools — an affine type system, a formal static model checker — that mathematically rule out a cycle before a binary is ever produced, accepting a real upfront cognitive cost in exchange for never seeing a non-deterministic deadlock in production at all.

The opposing position, common among engineers building databases and distributed transaction engines, treats that rigidity as unrealistic once the system's access patterns are dictated by arbitrary, runtime-generated requests it has no control over. Forcing a global static order onto queries nobody wrote in advance would collapse the engine into a serial bottleneck; better to accept that a bounded number of localized aborts and retries is the normal operating cost of real parallelism at that scale.

The resolution tracks whose hands are on the access pattern. If a system's lock-acquisition paths are written entirely by the team that owns it, static ordering is nearly free and the safer default. If the access pattern is generated by arbitrary external input at runtime — exactly the database case — no static analysis can order what hasn't been written yet, and runtime detection is the only strategy actually available. Ch 78's actor model sidesteps this entire dispute for a different reason: with no shared memory between actors in the first place, there is no lock to order and no cycle to detect — a structural elimination of the whole category this chapter catalogs, not a resolution of the debate within it.
