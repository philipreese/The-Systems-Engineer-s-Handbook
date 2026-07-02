# Table of Contents

Status: `[Stub]` = listed only | `[Draft]` = raw drafts exist | `[Complete]` = synthesized chapter committed

---

## Part I — Systems Thinking

*The mental models senior engineers use to evaluate designs. Everything in later parts builds on these concepts.*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 01 | What Engineering Actually Optimizes | [Complete] |
| Ch 02 | Complexity Is the Enemy | [Complete] |
| Ch 03 | Coupling and Cohesion | [Complete] |
| Ch 04 | Abstraction and Information Hiding | [Complete] |
| Ch 05 | Designing for Change | [Complete] |
| Ch 06 | Cost Models and Mechanical Sympathy | [Complete] |
| Ch 07 | Reliability as a Design Principle | [Complete] |
| Ch 08 | Local vs. Global Optimization | [Complete] |
| Ch 09 | Decision Frameworks for Trade-offs | [Complete] |

---

## Part II — Software Architecture

*Prerequisites: Part I*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 10 | Monolith vs. Service Decomposition | [Complete] |
| Ch 11 | Layered, Hexagonal, and Ports-and-Adapters Architecture | [Complete] |
| Ch 12 | Dependency Direction and Inversion | [Complete] |
| Ch 13 | Coupling and Cohesion at the Architecture Level | [Complete] |
| Ch 14 | Abstraction Layers: When to Add One | [Complete] |
| Ch 15 | API Surface Design: What to Expose, What to Hide | [Complete] |
| Ch 16 | Versioning and Backward Compatibility | [Complete] |
| Ch 17 | Synchronous vs. Asynchronous Communication | [Complete] |
| Ch 18 | Data Ownership Boundaries | [Complete] |

---

## Part III — API Design

*Prerequisites: Part I, Part II*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 19 | REST vs. RPC vs. Event-Driven | [Complete] |
| Ch 20 | Resource Modeling | [Complete] |
| Ch 21 | Error Handling Contracts | [Complete] |
| Ch 22 | Idempotency | [Complete] |
| Ch 23 | Pagination and Streaming | [Complete] |
| Ch 24 | Authentication and Authorization Boundaries | [Complete] |
| Ch 25 | Internal vs. External API Design | [Complete] |
| Ch 26 | FFI and Native Binding Design | [Complete] |

---

## Part IV — Code Organization

*Prerequisites: Part I*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 27 | File and Module Structure | [Complete] |
| Ch 28 | Naming Conventions and When They Matter | [Complete] |
| Ch 29 | When to Split Files vs. Keep Together | [Complete] |
| Ch 30 | Comments: What to Comment, What Not To | [Complete] |
| Ch 31 | When Abstractions Help vs. When They Obscure | [Complete] |
| Ch 32 | Error Handling: Typed Errors vs. Exceptions vs. Result Types | [Complete] |
| Ch 33 | When to Write Unsafe or Low-Level Code | [Complete] |

---

## Part V — Testing Strategy

*Prerequisites: Part I, Part IV*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 34 | The Testing Pyramid | [Complete] |
| Ch 35 | What Belongs at Each Layer | [Complete] |
| Ch 36 | When to Mock vs. Use Real Dependencies | [Complete] |
| Ch 37 | Fixture-Based Testing | [Complete] |
| Ch 38 | Property-Based Testing | [Complete] |
| Ch 39 | When Not to Test | [Complete] |
| Ch 40 | Test Naming and Structure | [Complete] |
| Ch 41 | Coverage: What It Measures and What It Doesn't | [Complete] |

---

## Part VI — Engineering Process

*Prerequisites: Part I*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 42 | Issue Tracking: What Makes a Good Issue | [Complete] |
| Ch 43 | Issue as Tracking Unit vs. PR as Review Unit | [Complete] |
| Ch 44 | Milestone and Phase Planning | [Complete] |
| Ch 45 | Architecture Decision Records (ADRs) | [Complete] |
| Ch 46 | Spec-First Development | [Complete] |
| Ch 47 | Code Review | [Complete] |
| Ch 48 | Technical Debt | [Complete] |
| Ch 49 | Process Overhead: The Value Threshold | [Complete] |

---

## Part VII — Git and Delivery

*Prerequisites: Part I, Part VI*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 50 | Branching Strategies | [Complete] |
| Ch 51 | Commit Message Conventions | [Complete] |
| Ch 52 | Squash vs. Preserve History | [Complete] |
| Ch 53 | Branch Naming and Lifecycle | [Complete] |
| Ch 54 | Force Push: When It's Acceptable | [Complete] |
| Ch 55 | Merge vs. Rebase | [Complete] |
| Ch 56 | Tagging and Release Markers | [Complete] |
| Ch 57 | What Belongs in CI (and What Doesn't) | [Complete] |
| Ch 58 | Fail-Fast vs. Fail-Safe Pipeline Design | [Complete] |
| Ch 59 | Caching Strategy in CI | [Complete] |
| Ch 60 | Matrix Builds | [Complete] |
| Ch 61 | Release Automation | [Complete] |
| Ch 62 | Environment Promotion | [Complete] |
| Ch 63 | Toolchain and Dependency Management | [Complete] |

---

## Part VIII — Documentation

*Prerequisites: Part I, Part IV*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 64 | What to Document vs. What to Leave to the Code | [Complete] |
| Ch 65 | README vs. Spec vs. ADR vs. Inline Comment | [Complete] |
| Ch 66 | Keeping Documentation Honest | [Complete] |
| Ch 67 | API Documentation: What Consumers Need | [Complete] |
| Ch 68 | Runbooks and Operational Documentation | [Complete] |

---

## Part IX — Observability

*Prerequisites: Part I, Part II*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 69 | Logging: What to Log and at What Level | [Complete] |
| Ch 70 | Metrics vs. Logs vs. Traces | [Complete] |
| Ch 71 | Alerting: Signal vs. Noise | [Complete] |
| Ch 72 | Distributed Tracing | [Stub] |
| Ch 73 | Error Budgets and SLOs | [Stub] |

---

## Part X — Concurrency and Parallelism

*Prerequisites: Part I, Part VI (cost models)*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 74 | Shared State vs. Message Passing | [Stub] |
| Ch 75 | Locks: When to Use Them | [Stub] |
| Ch 76 | Async vs. Threads vs. Processes | [Stub] |
| Ch 77 | Deadlock, Livelock, and Starvation | [Stub] |
| Ch 78 | The Actor Model | [Stub] |

---

## Part XI — Security

*Prerequisites: Part I, Part III*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 79 | Threat Modeling | [Stub] |
| Ch 80 | Defense in Depth | [Stub] |
| Ch 81 | Input Validation | [Stub] |
| Ch 82 | Authentication vs. Authorization | [Stub] |
| Ch 83 | Secrets Management | [Stub] |
| Ch 84 | Dependency Supply Chain Risk | [Stub] |

---

## Part XII — Performance

*Prerequisites: Part I (cost models), Part II, Part X*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 85 | When to Optimize (and When Not To) | [Stub] |
| Ch 86 | Profiling-First | [Stub] |
| Ch 87 | Latency vs. Throughput Trade-offs | [Stub] |
| Ch 88 | Caching: Layers, Invalidation, Failure Modes | [Stub] |
| Ch 89 | Data Structure and Algorithm Selection in Practice | [Stub] |
| Ch 90 | The Cost of Abstraction | [Stub] |

---

## Appendices

| Appendix | Title | Status |
|----------|-------|--------|
| A | Decision Frameworks | [Stub] |
| B | Common Engineering Smells | [Stub] |
| C | Architecture Patterns Catalog | [Stub] |
| D | Full Glossary | [Stub] |
