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
| Ch 23 | Pagination and Streaming | [Stub] |
| Ch 24 | Authentication and Authorization Boundaries | [Stub] |
| Ch 25 | Internal vs. External API Design | [Stub] |
| Ch 26 | FFI and Native Binding Design | [Stub] |

---

## Part IV — Code Organization

*Prerequisites: Part I*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 27 | File and Module Structure | [Stub] |
| Ch 28 | Naming Conventions and When They Matter | [Stub] |
| Ch 29 | When to Split Files vs. Keep Together | [Stub] |
| Ch 30 | Comments: What to Comment, What Not To | [Stub] |
| Ch 31 | When Abstractions Help vs. When They Obscure | [Stub] |
| Ch 32 | Error Handling: Typed Errors vs. Exceptions vs. Result Types | [Stub] |
| Ch 33 | When to Write Unsafe or Low-Level Code | [Stub] |

---

## Part V — Testing Strategy

*Prerequisites: Part I, Part IV*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 34 | The Testing Pyramid | [Stub] |
| Ch 35 | What Belongs at Each Layer | [Stub] |
| Ch 36 | When to Mock vs. Use Real Dependencies | [Stub] |
| Ch 37 | Fixture-Based Testing | [Stub] |
| Ch 38 | Property-Based Testing | [Stub] |
| Ch 39 | When Not to Test | [Stub] |
| Ch 40 | Test Naming and Structure | [Stub] |
| Ch 41 | Coverage: What It Measures and What It Doesn't | [Stub] |

---

## Part VI — Engineering Process

*Prerequisites: Part I*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 42 | Issue Tracking: What Makes a Good Issue | [Stub] |
| Ch 43 | Issue as Tracking Unit vs. PR as Review Unit | [Stub] |
| Ch 44 | Milestone and Phase Planning | [Stub] |
| Ch 45 | Architecture Decision Records (ADRs) | [Stub] |
| Ch 46 | Spec-First Development | [Stub] |
| Ch 47 | Code Review | [Stub] |
| Ch 48 | Technical Debt | [Stub] |
| Ch 49 | Process Overhead: The Value Threshold | [Stub] |

---

## Part VII — Git and Delivery

*Prerequisites: Part I, Part VI*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 50 | Branching Strategies | [Stub] |
| Ch 51 | Commit Message Conventions | [Stub] |
| Ch 52 | Squash vs. Preserve History | [Stub] |
| Ch 53 | Branch Naming and Lifecycle | [Stub] |
| Ch 54 | Force Push: When It's Acceptable | [Stub] |
| Ch 55 | Merge vs. Rebase | [Stub] |
| Ch 56 | Tagging and Release Markers | [Stub] |
| Ch 57 | What Belongs in CI (and What Doesn't) | [Stub] |
| Ch 58 | Fail-Fast vs. Fail-Safe Pipeline Design | [Stub] |
| Ch 59 | Caching Strategy in CI | [Stub] |
| Ch 60 | Matrix Builds | [Stub] |
| Ch 61 | Release Automation | [Stub] |
| Ch 62 | Environment Promotion | [Stub] |
| Ch 63 | Toolchain and Dependency Management | [Stub] |

---

## Part VIII — Documentation

*Prerequisites: Part I, Part IV*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 64 | What to Document vs. What to Leave to the Code | [Stub] |
| Ch 65 | README vs. Spec vs. ADR vs. Inline Comment | [Stub] |
| Ch 66 | Keeping Documentation Honest | [Stub] |
| Ch 67 | API Documentation: What Consumers Need | [Stub] |
| Ch 68 | Runbooks and Operational Documentation | [Stub] |

---

## Part IX — Observability

*Prerequisites: Part I, Part II*

| Chapter | Title | Status |
|---------|-------|--------|
| Ch 69 | Logging: What to Log and at What Level | [Stub] |
| Ch 70 | Metrics vs. Logs vs. Traces | [Stub] |
| Ch 71 | Alerting: Signal vs. Noise | [Stub] |
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
