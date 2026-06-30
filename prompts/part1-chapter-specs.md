# Part I — Chapter Specifications

Pre-filled special instructions for each Part I chapter. When generating a draft, copy the
block for the chapter you're writing into the `[SPECIAL INSTRUCTIONS]` field of the
chapter-generation prompt.

The instructions do three things:
1. State the chapter's specific role in the handbook's argument
2. Name the key sources, systems, or concepts to anchor the content
3. Explicitly mark what NOT to cover — because those topics belong to another chapter

---

## Ch 01 — What Engineering Actually Optimizes

```
This chapter establishes the foundational mental model for all later chapters: engineering is
multi-objective optimization under constraint. Do not rush to abstractions. Build the
reasoning model first — name the axes (latency, throughput, reliability, cost of change,
developer velocity), show how they trade off against each other, and explain why making
optimization targets explicit matters more than choosing the right ones.

Anchor in real systems: PostgreSQL (cost of change in schema design), Redis (execution
optimization trade-offs), Nagle's Algorithm (latency vs. throughput), Erlang/OTP vs.
SQLite (MTBF vs. MTTR paradigm).

Do NOT cover in depth: coupling/cohesion (Ch 03), abstraction layers (Ch 04), local vs.
global optimization (Ch 08). Mention these briefly and flag them as "expanded in later
chapters."
```

---

## Ch 02 — Complexity Is the Enemy

```
This chapter's core argument comes from Moseley & Marks ("Out of the Tar Pit", 2006):
complexity is the single root cause of most software failures. Distinguish precisely between
essential complexity (inherent to the problem domain — cannot be eliminated) and accidental
complexity (introduced by the engineering solution — can be reduced). Give engineers a
concrete way to identify which type they are dealing with.

The three sources of complexity are: state, control flow, and code volume. Cover each.

Anchor in real systems: SQLite (deliberate simplicity as a design principle — single file,
no server, no network), early Unix philosophy (small sharp tools over monolithic programs),
and one counterexample showing where complexity was allowed to accumulate (a specific
system, not a hypothetical).

Do NOT cover: coupling and cohesion as formal concepts (Ch 03), abstraction design (Ch 04),
when to decompose a system (Part II). This chapter names the enemy; later chapters deal with
specific weapons against it.
```

---

## Ch 03 — Coupling and Cohesion

```
Define these terms precisely — most engineers use them loosely and conflate them. Coupling
is about dependencies between components; cohesion is about the internal consistency of a
single component. Both matter, but for different reasons and at different times.

Cover coupling formally: distinguish afferent coupling (how many things depend on this
component) from efferent coupling (how many things this component depends on), and
introduce the connascence taxonomy (at minimum: name, type, value, execution order, timing)
as a vocabulary for describing coupling strength.

Include a practical method for measuring or identifying high coupling — not just theory.

Anchor in real systems: Unix pipes (loose coupling through a minimal interface), a
microservices example where intended decoupling became actual coupling through shared
database schemas or implicit API contracts.

Do NOT cover: dependency inversion as an architectural pattern (Part II Ch 12), API surface
design decisions (Part II Ch 15), module file structure (Part IV Ch 27). This chapter
defines the concepts; later chapters apply them at specific scales.
```

---

## Ch 04 — Abstraction and Information Hiding

```
The core distinction in this chapter is Parnas's information hiding (1972) vs. simple
encapsulation. Information hiding is about hiding the *decision* — the design choice that
is likely to change. Encapsulation merely hides implementation. These are not the same thing,
and conflating them leads to abstractions that feel correct but leak under change.

Cover Spolsky's Law of Leaky Abstractions: every non-trivial abstraction leaks. The question
is not whether it will leak, but when, under what conditions, and how badly.

The wrong abstraction is worse than no abstraction — this is the failure mode to dwell on.
Premature abstraction creates coupling to an interface that doesn't fit the problem yet.

Anchor in real systems: the POSIX file descriptor abstraction (what it hides, what it
deliberately exposes, and why it has survived 50 years), the Linux VFS layer (information
hiding at the OS level), an ORM as a case study in an abstraction that leaks badly under
production query load.

Do NOT cover: API versioning strategies (Part II Ch 16), module and file structure (Part IV
Ch 27), when to split a service (Part II Ch 10). This chapter is about the concept of
abstraction itself — not about where to apply it architecturally.
```

---

## Ch 05 — Designing for Change

```
The core question is: what should be stable, and what should be easy to change? These are
not the same set of things. A good design stabilizes the right things (interfaces, contracts,
data shapes) and makes the right things easy to change (implementations, algorithms,
dependencies behind an interface).

Cover the open/closed principle honestly — not as dogma, but as a useful lens with real
limits. Cover why designing for change is different from "future-proofing": designing for
change means identifying specific axes of variation and making those cheap; future-proofing
often means speculating about unspecified future requirements and paying complexity now
for benefits that may never arrive.

Anchor in real systems: Git's content-addressable object store (immutable core that has
survived 20 years of new features), the Linux syscall ABI (stability as a design commitment,
not an accident), PostgreSQL's wire protocol stability across major versions.

Do NOT cover: versioning strategies for APIs (Part II Ch 16), branching strategies (Part VII
Ch 50), ADRs as a process (Part VI Ch 45). This chapter is about the structural design
principle; the process and tooling live elsewhere.
```

---

## Ch 06 — Cost Models and Mechanical Sympathy

```
This is the most concrete chapter in Part I. The goal is to give engineers a working mental
model of what computation actually costs at the hardware level, so architectural decisions
stop being purely abstract and start being grounded in physical reality.

Include actual latency numbers — these are the ones engineers should memorize:
  L1 cache: ~1ns | L2 cache: ~4ns | L3 cache: ~40ns
  Main memory (RAM): ~100ns | SSD random read: ~100μs
  Network round trip (same datacenter): ~500μs–1ms | Cross-region: ~100ms

Cover Martin Thompson's "mechanical sympathy" concept: you write better systems when you
understand what the hardware is actually doing.

Explain why these numbers matter architecturally: why Redis is fast (in-memory, avoids disk
and network for reads), why Kafka is fast (sequential I/O amortizes seek cost), why
PostgreSQL MVCC is designed the way it is (avoids lock contention at the cost of storage and
vacuum overhead), why a cache miss in a tight loop can dominate performance more than
algorithmic complexity.

Do NOT cover: profiling strategy and workflow (Part XII Ch 86), caching layer design (Part
XII Ch 88), algorithm selection in practice (Part XII Ch 89). Those chapters apply this
foundation — this chapter establishes it.
```

---

## Ch 07 — Reliability as a Design Principle

```
Ch 01 introduced MTBF vs. MTTR as a paradigm choice. This chapter goes deeper: reliability
is an emergent property of system design, not a feature you add. Cover what this means
concretely.

Start with the failure mode taxonomy — not all failures are equal:
  1. Crash (visible, recoverable)
  2. Corruption (silent, often undetectable until late)
  3. Wrong answer (most dangerous — system appears healthy while producing incorrect output)
Cover these in order of danger. Wrong answers are worse than crashes.

Cover the "fail fast" principle: why detecting and crashing on a bad state early is safer
than attempting to continue. Cover partial failure in distributed systems — the scenario
where some components succeed and others fail simultaneously, producing a state that is
neither success nor failure.

Cover the CAP theorem honestly: do not oversimplify into "pick two." Cover what the
theorem actually says and what it means in practice for system design decisions.

Anchor in real systems: PostgreSQL WAL (how write-ahead logging achieves durability without
blocking on every fsync), ZooKeeper / etcd (consensus and its cost), Kafka's log-based
durability model.

Do NOT cover: error budgets and SLOs (Part IX Ch 73), distributed tracing (Part IX Ch 72),
alerting strategy (Part IX Ch 71). Those belong to the observability part. This chapter is
about designing reliability in, not about detecting when it fails.
```

---

## Ch 08 — Local vs. Global Optimization

```
Ch 01 mentioned this as a concept and flagged it for this chapter. Now expand it fully.

The core argument: a system is not the sum of its components. Optimizing individual
components independently does not produce a globally optimal system, and frequently makes
it worse. This is a systemic property with formal underpinnings.

Cover these frameworks:
  - Little's Law (L = λW): the relationship between throughput, arrival rate, and queue
    depth. This gives a formal basis for why local latency improvements can degrade system
    throughput.
  - Theory of Constraints (Goldratt): a system's performance is bounded by its bottleneck.
    Optimizing non-bottleneck components wastes effort and may worsen the bottleneck.
  - Conway's Law: organizations ship systems that mirror their communication structure.
    This is local vs. global optimization applied to team structure — local team autonomy
    creates global system coupling.

Anchor in real systems: a microservices architecture where each service reduces its own
latency but increases cross-service call fan-out, resulting in higher end-to-end latency
(the "distributed monolith" failure mode). Database connection pooling as a global resource
that individual services treat as local.

Do NOT cover: profiling individual components (Part XII), caching strategy (Part XII),
distributed tracing as an observability tool (Part IX). This chapter is about system-level
reasoning, not the tools used to observe a running system.
```

---

## Ch 09 — Decision Frameworks for Trade-offs

```
This closes Part I. The previous eight chapters gave the reader vocabulary and mental models.
This chapter gives them a process for applying those models when making real decisions under
uncertainty, time pressure, and incomplete information.

Cover these frameworks concretely:
  1. Reversibility × blast radius matrix: classify decisions by how hard they are to undo
     and how much damage a wrong call causes. High-irreversibility + high-blast-radius
     decisions deserve deliberation; low × low decisions should be made quickly and moved on.
  2. When to defer a decision vs. make it now: deferral is legitimate when making the
     decision now requires information you don't have yet, and waiting is cheap. Deferral
     is costly when the absence of a decision is itself a decision that accumulates interest.
  3. The Cynefin framework applied to engineering: simple (best practice exists), complicated
     (analysis required, right answer exists), complex (probe-sense-respond, no single right
     answer), chaotic (act first, stabilize). Most architectural decisions are complicated,
     not complex — there is a defensible right answer, it just requires analysis.

Cover the cost of indecision: analysis paralysis is a failure mode with real consequences.
Teams that cannot commit to an architecture ship nothing.

Briefly introduce ADRs (Architecture Decision Records) as a lightweight tool for recording
what was decided, why, and what alternatives were rejected. Do NOT go deep on the ADR
process — that belongs in Part VI Ch 45. Here, one paragraph establishing their purpose
is enough.

Do NOT write this as a generic "decision making" chapter. Every example must be an
engineering decision — technology choices, API design, data model design, deployment
strategy. Abstract management frameworks without engineering anchoring do not belong here.
```
