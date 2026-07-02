# Ch 76 — Async vs. Threads vs. Processes

**Prerequisites:** [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md), [Cost Models and Mechanical Sympathy](../part01-systems-thinking/ch06-cost-models-and-mechanical-sympathy.md) (latency hierarchy, context-switch cost), [Shared State vs. Message Passing](ch74-shared-state-vs-message-passing.md), [Locks: When to Use Them](ch75-locks-when-to-use-them.md)

**New vocabulary introduced:** concurrency, parallelism, CPU-bound work, I/O-bound work

**Key takeaways:**
- Ch 74 and Ch 75 answered how concurrent units coordinate. This chapter answers a separate question: what actually runs the work. The two axes combine freely — an async runtime can still need locks around shared state, and threads can communicate exclusively through channels — and confusing them produces a "threads vs. async" debate when the real disagreement is about coordination.
- **Concurrency** is a property of a program's structure — making progress on more than one unit of work in overlapping time — and requires no particular hardware. **Parallelism** is a property of execution — more than one unit of work actually running at the same physical instant — and requires multiple cores. A program can be one, both, or neither.
- [Strong Recommendation] The selection test is not "which model is fastest" but "what is the program actually waiting for." CPU-bound work needs true parallelism (threads or processes across cores) to go faster; I/O-bound work needs concurrency without needing more cores at all, and async is often strictly cheaper for it.
- Using threads for I/O-bound work pays context-switch and memory overhead that buys nothing. Using async for CPU-bound work is the worse failure: one long computation blocks the single event loop and stalls every other concurrent task sharing it, not just the one that caused it.
- Real runtimes make this trade-off visible rather than resolving it: Python's GIL means OS threads don't buy CPU parallelism at all; Node.js commits fully to a single-threaded event loop for I/O-bound work; Go's goroutines are a deliberate M:N hybrid, scheduled by the runtime onto a small pool of OS threads, blurring the thread/async line by design.

---

Chapters 74 and 75 established how concurrent units coordinate access to shared data — through synchronization or through messages. This chapter asks a different, independent question: what actually executes the work. A team arguing over "threads versus async" while the real disagreement is about lock granularity is solving the wrong problem, and vice versa; the two decisions combine freely, and this chapter concerns only the execution vehicle.

### Decision: Choose the Execution Vehicle — Process, Thread, or Async Task

**What it is:** Which physical or runtime-managed unit actually carries out concurrent work: an OS process, an OS thread, or an async task scheduled cooperatively by a runtime.

**Why it exists:** Waiting for network or disk I/O costs orders of magnitude more time than executing a CPU instruction (Ch 06's latency hierarchy). A vehicle built to hold that wait — a full OS thread, with its own stack and kernel scheduling overhead — is expensive to spin up by the thousands merely to sit idle. A vehicle built to avoid holding a wait — an async task multiplexed onto a small number of threads — is cheap per unit but buys no parallelism on its own. Processes add a further, orthogonal property: a genuinely separate address space, at a further cost in creation time and communication overhead.

**Options:**
- **OS process** — the operating system's unit of resource ownership: its own virtual address space, file descriptors, and security context. Communicates with other processes only through explicit IPC (pipes, sockets, shared memory).
- **OS thread** — an execution context within a process, sharing its address space with every other thread in that process, scheduled preemptively by the kernel.
- **Async task / coroutine** — a lightweight, runtime-managed unit that cooperatively yields control back to an event loop at an explicit suspension point (typically a non-blocking I/O call) instead of being preemptively interrupted.

**Trade-offs:**

| | Process | Thread | Async task |
|---|---|---|---|
| Isolation | Full — separate address space | None — shares process memory | None — shares process memory |
| Creation / per-unit cost | Highest | Moderate (megabytes of stack, kernel-scheduled) | Lowest (kilobytes, user-space scheduled) |
| Scheduling | Preemptive (kernel) | Preemptive (kernel) | Cooperative (runtime) |
| True CPU parallelism | Yes, across cores | Yes, across cores | No, unless backed by an explicit thread pool |
| Failure containment | A crash stays inside the process | A fatal error in one thread can take down the whole process | Same as thread — shares the host process's fate |

**When to choose each:** [Strong Recommendation] A process when fault isolation or a hard security boundary matters more than communication cost — components that should fail independently, or that run untrusted or third-party code. A thread when CPU-bound work must scale across cores and the cost of sharing memory (Ch 75's locking discipline) is acceptable. An async task when the workload is dominated by waiting on I/O and the goal is holding many concurrent operations cheaply, not computing faster.

**Common failure modes:** Thread exhaustion under I/O saturation — a reverse proxy built thread-per-request spawns thousands of OS threads under a burst of slow client connections, and the kernel spends its cycles context-switching between mostly-idle threads instead of moving data, collapsing throughput. Cascading shared-process collapse — one edge-case request triggers a fatal, unmanaged error inside a single thread of a multithreaded server, and the kernel tears down the entire process, disconnecting every other, unrelated client connection along with it.

**Example:** PostgreSQL forks a fresh backend process per client connection specifically so that a fault triggered by one client's query terminates only that one backend, leaving the rest of the cluster untouched — a direct trade of per-connection overhead for blast-radius containment. Node.js commits to the opposite end for a different reason: a single-threaded event loop handling thousands of concurrent network connections with a fraction of the memory a thread-per-connection model would need, because the workload it targets is overwhelmingly I/O-bound.

### Decision: Match the Execution Model to the Workload's Actual Bottleneck

**What it is:** Whether the dominant limiting resource in a given workload is the processor itself (CPU-bound) or time spent waiting on an external resource such as network or disk (I/O-bound) — and choosing the execution vehicle accordingly.

**Why it exists:** CPU-bound and I/O-bound workloads fail differently when given the wrong vehicle. Adding concurrency to CPU-bound work without adding execution cores cannot make it faster — the processor is already saturated, and more scheduling only adds overhead. Adding OS threads to I/O-bound work adds memory and context-switch cost without reducing the time spent waiting, since the threads spend nearly all their existence blocked rather than computing.

**Options:** Threads or processes for CPU-bound work, since only they provide true parallelism across cores. Async execution for I/O-bound work, since it holds many concurrent waits cheaply without needing more cores. Hybrid designs for workloads that are genuinely both — an async front end managing network connections while handing CPU-heavy work off to a thread or process pool.

**Trade-offs:** Threads or processes on CPU-bound work convert available cores directly into throughput, but bring no benefit — only overhead — to a workload that's mostly waiting. Async on I/O-bound work holds enormous numbers of concurrent operations cheaply, but provides no computational speedup and actively fails if a CPU-heavy operation is ever run on it directly. Hybrid designs match each part of the system to its actual bottleneck, at the cost of running two different concurrency models side by side.

**When to choose each:** Threads or processes when computation genuinely dominates — cryptographic operations, image or video processing, numerical simulation. Async when waiting genuinely dominates — a gateway or proxy juggling thousands of mostly-idle connections. Hybrid whenever a system has both kinds of work in different places, which is the common case in practice, not the exception.

**Common failure modes:** Using threads for I/O-bound work pays unnecessary context-switch and memory cost for no gain — the failure is wasteful, but bounded. Using async for CPU-bound work is the worse failure by construction: a single long-running computation — even something as ordinary as a cryptographic signature check on one request — occupies the one event-loop thread directly, and every other concurrent task queued behind it stalls completely until that computation finishes, turning one slow request into an outage for every unrelated one sharing the loop.

**Example:** Python's reference implementation ships a Global Interpreter Lock that restricts bytecode execution to one core at a time regardless of how many OS threads a program spawns — threads remain useful there for overlapping I/O, but CPU-bound Python work needs separate processes to get real parallelism at all. Go takes a third path: goroutines are lightweight, cooperatively-scheduled-looking tasks that the Go runtime itself maps onto a small pool of OS threads (an M:N scheduler), so a goroutine that blocks on I/O yields without blocking the underlying thread, while CPU-bound goroutines can still run genuinely in parallel once enough OS threads are available — deliberately blurring the line this chapter otherwise draws between async and threads.

### Why Smart Engineers Disagree on Hybrid (M:N) Runtimes

The disagreement here isn't CPU-bound-vs-I/O-bound — that test is not seriously contested. It's whether a runtime should hide the process/thread/async distinction entirely behind a scheduler, the way Go's goroutines do.

One position treats Go's M:N model as the right synthesis: forcing engineers to manually choose between explicit `async`/`await` syntax and raw OS threads is unnecessary cognitive overhead. The Go runtime intercepts a blocking call, parks the goroutine, and keeps the underlying OS thread busy with other work — giving engineers the low overhead of async concurrency with the linear, blocking-style code of threads, without asking them to track which model they're in.

The opposing position — closer to Rust's, C++'s, or Node.js's explicit models — treats a hidden scheduler as its own accidental complexity. It bakes a permanent, non-trivial runtime into every binary, which rules it out for embedded targets or lightweight FFI plugins, and when an M:N scheduler's performance degrades, debugging it requires understanding the scheduler's own internal heuristics rather than just the program's code — the execution vehicle stopped being visible in the syntax the moment the runtime took over choosing it.

Neither side disputes the underlying CPU-bound/I/O-bound test; they disagree about whether making the execution vehicle implicit is worth the opacity it introduces when something goes wrong. A hybrid runtime optimizes for uniform code and low per-task overhead across a large, varied codebase; an explicit model optimizes for a cost that's always visible directly in the code, at the price of engineers having to choose it correctly themselves.
