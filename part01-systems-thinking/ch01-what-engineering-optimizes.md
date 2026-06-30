# Chapter 1 — What Engineering Actually Optimizes

**Prerequisites:** None. This is the entry point.

**New vocabulary introduced:** optimization target, accidental complexity, essential complexity, MTBF, MTTR, cost of change, optimization target drift

**Key takeaways:**
- Every real system optimizes multiple objectives simultaneously, whether or not those objectives are named
- The objectives are frequently in tension; improving one degrades another
- Most architectural disagreements are really disagreements about which objective to prioritize, not about technical correctness
- Cost of change is the dominant long-term objective for most systems — more so than runtime performance
- Implicit optimization targets cause more damage than wrong explicit ones

---

## Purpose

Engineering is usually described as building systems that work. That framing is incomplete in a way that causes most real architectural mistakes.

A system that "works" at 100 requests per day may not work at 100,000. A system built to minimize latency may become unmaintainable in two years. A system designed by an infrastructure team for reliability may be unusable for a product team that needs to ship weekly.

The question is not whether a system works. The question is: *what is it actually optimizing for?*

This chapter establishes the primary axes along which engineering decisions are made, how those axes trade off against each other, and why making the optimization targets explicit is more important than choosing the right ones.

---

## The Primary Optimization Axes

Every non-trivial engineering decision is implicitly a point in a space defined by these objectives:

| Objective | What it means |
|-----------|--------------|
| **Latency** | Time to complete a single unit of work |
| **Throughput** | Volume of work completed per unit of time |
| **Reliability (MTBF)** | How rarely the system fails |
| **Resilience (MTTR)** | How quickly the system recovers when it does fail |
| **Cost of change** | How expensive it is to modify the system over time |
| **Operational burden** | How much effort the system requires to run and maintain |
| **Developer velocity** | How fast a team can deliver new behavior |

No system optimizes all of these simultaneously. Every strong position on one axis is implicitly a weak position on at least one other. The engineers who built the system chose a weighting — often without stating it. The engineers who maintain it are bound by that choice.

---

## Latency vs. Throughput

**What it is:** The tension between minimizing the time to complete one unit of work and maximizing the total volume of work completed over a period of time.

**Why it exists:** Hardware resources — CPU caches, network bandwidth, disk I/O — operate most efficiently when work is batched. Every operation carries fixed overhead: context switches, packet headers, I/O interrupts. Minimizing latency means paying this fixed cost immediately for each request. Maximizing throughput means waiting to amortize it across many requests.

**Options:**
1. **Immediate processing** — handle each input the moment it arrives, unbuffered
2. **Batch processing** — queue inputs until a time or size threshold is met, then process together
3. **Adaptive batching** — batch under load, process immediately when idle (Nagle's Algorithm, Kafka's `linger.ms`)

**Trade-offs:**

| Approach | Latency | Throughput | Complexity |
|----------|---------|-----------|------------|
| Immediate | Lowest possible | Lower — fixed costs paid per request | Low |
| Batch | Inflated for early requests | Higher — fixed costs amortized | Low |
| Adaptive | Near-immediate when idle | Near-optimal under load | Moderate |

**When to choose each:**
- *Immediate processing:* When latency has hard deadlines — high-frequency trading, real-time audio, human-interactive sessions. The cost of a missed deadline exceeds the cost of reduced throughput.
- *Batch processing:* When data volume exceeds continuous processing capacity, or when I/O is the bottleneck — analytics pipelines, log aggregation, bulk database inserts.
- *Adaptive:* When load is variable and both latency and throughput matter — most general-purpose network services.

**Common failure modes:**
- **Buffer bloat:** Unbounded queues intended to maximize throughput fill during load spikes, causing latency to climb exponentially as requests queue behind each other. The system is fast on average and catastrophically slow in the tail.
- **Thrashing under microburst:** An interrupt-driven system hit with a burst of high-frequency requests spends all CPU cycles context-switching. Throughput collapses to near zero. The system looks healthy in metrics (it's responding to everything) but gets nothing done.

**Example:** Nagle's Algorithm in TCP batches small packets to reduce header overhead, increasing network throughput. Engineers explicitly disable it (via `TCP_NODELAY`) for SSH sessions, multiplayer games, and financial feeds — deliberately sacrificing throughput efficiency for perceptible responsiveness. The flag exists because both choices are correct in different contexts. **[Legitimate Trade-off]**

---

## MTBF vs. MTTR: Two Reliability Paradigms

**What it is:** A fundamental choice in how to approach failure. *Mean Time Between Failures* (MTBF) optimizes for preventing failure. *Mean Time To Recovery* (MTTR) accepts failure as inevitable and optimizes for recovering fast.

**Why it exists:** In sufficiently complex distributed systems, perfect reliability is mathematically unachievable — network partitions happen, hardware degrades, state spaces exceed what testing can cover. Engineers must decide whether to invest in preventing failures or in recovering from them quickly. Both are legitimate; they imply different architectures.

**Options:**
1. **High-assurance / prevent failure (MTBF):** Formal proofs, exhaustive test coverage, defensive programming, extensive static analysis. Minimize the probability of any single failure.
2. **Crash-only / recover fast (MTTR):** Accept that components will fail. Design for idempotency, aggressive timeouts, and fast external restart. Supervisors replace failed components before anyone notices.

**Trade-offs:**

| Approach | Failure frequency | Recovery behavior | Development cost | Failure character |
|----------|------------------|-------------------|-----------------|------------------|
| MTBF | Lower | Slow or undefined | High | Catastrophic when it finally fails |
| MTTR | Higher (micro-failures) | Fast by design | Lower | Frequent but bounded |

The MTBF trap: systems designed never to fail often lack recovery mechanisms for states that weren't supposed to exist. When those states occur anyway — and they do — the system has no graceful path. It fails hard.

The MTTR trap: without strict idempotency, crash-and-restart loops corrupt state. A process that restarts into the same malformed input will loop forever.

**When to choose each:**
- *MTBF:* When recovery is impossible or failure consequences are severe — SQLite (file corruption is unrecoverable), aviation control systems, database storage engines, financial ledgers. The cost of a single failure exceeds the cost of extensive prevention.
- *MTTR:* Stateless web fleets, background job processors, microservices where dropped requests are retried by clients. Fast recovery is cheaper than prevention and failure is bounded.

**Common failure modes:**
- **Poison pill loop (MTTR):** A supervisor restarts a process that crashes on a malformed message in a queue. The process restarts, reads the same message, crashes again. Without dead-letter queues and backoff, this loops indefinitely and looks like normal churn in metrics.
- **Defensive deadlock (MTBF):** A process catches an unexpected hardware fault and holds internal locks in a corrupted state rather than crashing. The process remains alive to the load balancer but serves 100% errors. This is worse than a crash.

**Example:** Erlang/OTP is the canonical MTTR system. Processes are allowed — encouraged — to crash. Supervisors detect and restart them in microseconds. The entire OTP framework is designed around the assumption that individual processes will fail, frequently. SQLite is the canonical MTBF system: its test suite achieves 100% branch coverage, and its crash handling is designed to prevent file corruption under any termination scenario. Both are correct for their domains. **[Legitimate Trade-off]**

### Why Smart Engineers Disagree on Reliability

Infrastructure engineers default to MTBF thinking: they've been paged at 2am because something failed that shouldn't have. Product engineers default to MTTR thinking: they need to ship fast and can't afford the overhead of exhaustive prevention. Neither is wrong. The disagreement is a difference in which failure consequence they've personally experienced as most painful.

---

## Cost of Change vs. Cost of Execution

**What it is:** The long-term behavior of a system is determined more by how expensive it is to modify than by how it performs at rest. Cost of execution (runtime performance) is visible and measurable. Cost of change is deferred and often invisible until it becomes catastrophic.

**Why it exists:** Systems spend most of their operational lifetime being changed, not just being run. Features are added, bugs are fixed, schemas are migrated, dependencies are upgraded. The engineers who maintain a system are rarely the engineers who designed it. If the system is expensive to change, it will be changed slowly, incorrectly, or not at all.

**Options:**
1. **Optimize for execution:** Tight coupling, clever implementations, performance-first design. Fast at runtime; expensive to modify.
2. **Optimize for change:** Strong interfaces, clear boundaries, modular design. May sacrifice some runtime efficiency for long-term flexibility.
3. **Optimize explicitly per component:** Critical paths get execution optimization; peripheral systems get change optimization.

**Trade-offs:**

| Approach | Runtime performance | Change velocity | Cognitive load | Risk per change |
|----------|--------------------|--------------|--------------| --------------|
| Execution-first | High | Slow | High (must understand internals to change anything) | High |
| Change-first | Adequate | Fast | Lower (boundaries contain the blast radius) | Low |
| Per-component | Variable | Variable | High (two mental models to maintain) | Variable |

**When to choose each:**
- *Execution-first:* Components where runtime performance is a hard constraint and change frequency is genuinely low — a custom memory allocator, a database storage engine, a network protocol parser.
- *Change-first:* Product systems, APIs, anything touched by multiple engineers. The frequency of change makes this the dominant cost axis.
- *Per-component:* Mature systems with identifiable hot paths. But do this deliberately, after profiling — not as a default.

**Common failure modes:**
- **"We can't change this without breaking five other services."** This is the symptom of a system that was optimized for execution at the expense of change. The coupling that made it fast now makes every modification expensive.
- **Accidental immortality:** A PostgreSQL schema designed for a previous product era becomes effectively immutable because schema migrations now cost days of engineering time and require downtime. The schema outlives the original reasoning behind it.
- **Git archaeology:** When change cost is high, engineers treat `git log` as the only documentation of why something is the way it is. Every change requires understanding a year of history before touching anything.

**Example:** The Git object store is content-addressed and append-only. Individual object writes are not especially fast. But the model's immutability makes the system extraordinarily cheap to change: new features can be built on the existing object model without risk of corrupting existing history. Git has survived 20 years of new features because its core data model was designed to be stable. **[Strong Recommendation: for most systems, optimize the cost of change over cost of execution until profiling identifies a specific runtime bottleneck]**

---

## Explicit vs. Implicit Optimization Targets

**What it is:** Whether the system's actual optimization objectives are stated and shared, or unspoken and inferred from the code.

**Why it exists:** Systems are built and maintained by multiple people over time. If the optimization targets are not explicit, each person infers them from what they see — and those inferences diverge. The result is a system that was once designed coherently but is now drifting toward multiple inconsistent objectives simultaneously.

**Options:**
1. **Explicit targets:** Documented SLOs, architecture decision records, stated performance budgets, named trade-offs.
2. **Implicit targets:** Engineers read the code and infer what was being optimized, or assume their own priorities are shared.

**Trade-offs:**
- *Explicit:* Requires upfront investment to articulate. Enables coherent decision-making across teams and across time. Forces engineers to name the real trade-off instead of pretending there isn't one.
- *Implicit:* Faster to start. Inevitably produces drift as different contributors optimize for different things without realizing it.

**When to choose each:** There is no legitimate case for implicit targets in a system maintained by more than one person. The "cost" of making targets explicit — an ADR, a section in a README, an SLO document — is negligible compared to the cost of the drift it prevents. **[Strong Recommendation: always make optimization targets explicit for any system that will outlive its first author]**

**Common failure modes:**
- **"We optimized latency and broke reliability."** No one decided to break reliability. They optimized for the metric they were measuring without writing down what they were willing to sacrifice.
- **The unkillable hotfix:** A configuration change or query hint introduced as a temporary fix during an incident becomes permanent because no one knows whether it's still needed or what would break if removed. The system now has an implicit objective — "don't touch the hint" — that isn't written anywhere.
- **"We don't know why this is fast, but don't touch it."** The most dangerous sentence in systems engineering. It means the optimization target was never explicit, a change produced an observed effect no one understood, and that effect has now been promoted to an untouchable constraint.

**Example:** Kubernetes clusters accumulate implicit targets through incident-driven changes: a node affinity rule added during a capacity crunch, a resource limit set during an OOM event, a pod disruption budget configured to stop a cascading failure. None of these are wrong individually. But after a year, the cluster is optimizing for a set of constraints that no one explicitly chose and no one can fully enumerate. New engineers make changes that violate these implicit constraints and cause incidents. The root cause is not the individual changes — it's the absence of explicit documentation of what the system is being asked to optimize. **[Strong Recommendation: treat incident-driven configuration changes as candidates for explicit documentation, not just silent fixes]**

---

## Why Smart Engineers Disagree

Engineering disagreements are almost never about facts. They are about which objective to prioritize, and different engineers reach different answers because they have been shaped by different experiences.

**The role-based weighting problem:** An infrastructure engineer who has been paged for 2am incidents defaults to reliability and operational simplicity. A product engineer under shipping pressure defaults to developer velocity and cost of change. A performance engineer who has watched a system become too slow to use defaults to execution efficiency. None of them is wrong about the costs they've seen. They are wrong to assume their weighting is universal.

**The YAGNI vs. future-proofing disagreement:** The most frequent architectural dispute between senior engineers is not *how* to build something, but *when* to pay for the complexity of building it right. Engineers who favor speed to market optimize for minimal accidental complexity today, accepting that a rewrite may be required if the product succeeds. Engineers who favor future-proofing optimize for avoiding that rewrite, accepting a higher upfront tax in operational complexity. Both positions are rational. The disagreement is an optimization mismatch between engineers prioritizing different time horizons.

**What this means in practice:** When an architectural debate feels intractable, stop arguing about the technical merits of each option. Name the objective each option is optimizing for. The disagreement usually dissolves — or becomes a cleaner argument about which objective should win in this context, which is a decision you can actually make.

---

## What This Chapter Establishes

Every subsequent chapter in this handbook is a more specific version of the same question: *what is being optimized, and what does that choice cost?*

When Part II covers monolith vs. microservices, the question is whether you're optimizing for deployment independence or for operational simplicity. When Part V covers the testing pyramid, the question is whether you're optimizing for confidence or for development speed. When Part X covers locks vs. message passing, the question is whether you're optimizing for throughput or for reasoning simplicity.

The axes introduced here — latency, throughput, reliability, resilience, cost of change, operational burden, developer velocity — recur throughout. The frameworks introduced here — explicit vs. implicit objectives, MTBF vs. MTTR, execution cost vs. change cost — are applied repeatedly.

The rest of the handbook assumes you have internalized this chapter's core claim: **engineering is multi-objective optimization under constraint, the objectives are often in tension, and most architectural mistakes are really failures to name what was being optimized.**

*Concepts expanded in later chapters: accidental vs. essential complexity (Ch 02), coupling and its measurement (Ch 03), abstraction leaks (Ch 04), local vs. global optimization (Ch 08).*
