# Ch 27 — File and Module Structure

**Prerequisites:** [Complexity Is the Enemy](../part01-systems-thinking/ch02-complexity-is-the-enemy.md), [Coupling and Cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md), [Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [Layered, Hexagonal, and Ports-and-Adapters Architecture](../part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md), [Dependency Direction and Inversion](../part02-software-architecture/ch12-dependency-direction-inversion.md)

**New vocabulary introduced:** package-by-layer, package-by-feature, god package

**Key takeaways:**
- A file tree is not storage — it is the physical enforcement mechanism for architectural boundaries.
- Package-by-feature is the correct default for application code; package-by-layer belongs in frameworks where the technical role *is* the domain.
- Ports belong with the domain. Adapters belong with infrastructure. Physical directory segregation lets the compiler enforce dependency direction.
- Circular package dependencies are almost always a sign of an incorrectly drawn architectural boundary, not a naming or tooling problem.
- A `utils/` or `common/` package is not an architectural layer — it is the absence of one.

---

Previous chapters established the principles of [coupling and cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md), [information hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md), [ports and adapters](../part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md), and [dependency direction](../part02-software-architecture/ch12-dependency-direction-inversion.md). This chapter answers the question those chapters explicitly deferred: once those principles are understood, how do they show up as directories and import relationships in a real codebase?

Naming within packages is covered in [Ch 28](ch28-naming-conventions-and-when-they-matter.md). Deciding when to split a single concept across multiple files is covered in [Ch 29](ch29-when-to-split-files-vs-keep-together.md).

---

### Package-by-Layer vs. Package-by-Feature

**What it is:** The primary organizing decision for a codebase's root structure — whether packages are grouped by technical role or by business capability.

**Why it exists:** When a business requirement changes, an engineer modifies code. The directory structure determines whether those modifications are localized to one package or scattered across the entire repository.

**Options:**

1. **Package-by-layer** — Directories named after technical roles: `controllers/`, `services/`, `repositories/`, `models/`.
2. **Package-by-feature** — Directories named after business capabilities: `orders/`, `billing/`, `inventory/`, each containing its own controller, service, and repository.
3. **Hybrid** — Feature packages for application code; genuinely generic infrastructure (logging, metrics, config) in their own explicitly named packages outside the feature tree.

**Trade-offs:**

| | Package-by-layer | Package-by-feature |
|---|---|---|
| Cohesion | High *technical* cohesion | High *domain* cohesion |
| Change locality | A single feature change touches many directories | A single feature change stays in one directory |
| Team ownership | Difficult — no directory maps to a team's domain | Natural — a team owns a directory subtree |
| Cross-feature leakage | High — shared directories blur domain boundaries | Low — features are physically isolated |

Package-by-layer maximizes technical cohesion: all HTTP parsing logic is in one folder. It destroys domain cohesion: every business feature is scattered across every technical directory. The cost surfaces as shotgun surgery — a single new feature requires modifying `controllers/`, `services/`, `repositories/`, `models/`, and `tests/`. Six directories; zero of which map to the feature itself.

Package-by-feature reverses this. All billing logic — controller, service, repository — lives in `billing/`. The feature has a home. The trade-off is minor duplication of technical boilerplate across feature packages, which is almost always the cheaper cost.

**When to choose each:**

[Consensus] Choose **package-by-feature** for business applications, product code, and any codebase organized around domain capabilities. Industry practice has broadly converged on this.

[Strong Recommendation] Choose **package-by-layer** only for framework code, generic platform libraries, or low-level infrastructure where the technical role genuinely *is* the primary domain — building a query parser, a connection pool, or a protocol implementation. In those contexts, `parsing/`, `execution/`, and `mapping/` accurately describe what evolves independently. The hybrid — feature packages plus a small set of explicitly named infrastructure packages — is the dominant structure in mature production systems.

**Common failure modes:** In a package-by-layer structure, `services/` accumulates 150 files spanning 20 unrelated business domains. Navigating it requires holding the entire system in memory. Every new developer adds their service alongside services it has nothing to do with. The structure describes the technical stack; it communicates nothing about what the system does.

**Example:** Idiomatic Go rejects package-by-layer for application code. A repository with `models/`, `controllers/`, and `utils/` top-level directories is a documented anti-pattern in the Go community, repeated by Go's own maintainers. Go's package-level visibility rules require exporting everything from a shared package — a `models/` package must export all its types, destroying information hiding. When `User` inevitably references `Order` and `Order` references `User`, the result is a circular import the compiler refuses to build, forcing the structural failure into the open at compile time.

---

### Physical Placement of Ports and Adapters

**What it is:** The literal directory mapping for the hexagonal architecture from [Ch 11](../part02-software-architecture/ch11-layered-hexagonal-ports-adapters.md) — where interfaces (ports) live on disk, and where the infrastructure that implements them lives.

**Why it exists:** [Dependency inversion](../part02-software-architecture/ch12-dependency-direction-inversion.md) requires that the domain not depend on infrastructure. If the domain package and its PostgreSQL adapter share a directory, an engineer will eventually import a PostgreSQL-specific type directly into business logic. Nothing in the toolchain will stop them. Physical package segregation transforms the dependency rule from a code-review checklist item into a compiler constraint.

**Options:**

1. **Core-owned ports** — The domain package contains the interfaces it needs; adapter packages implement those interfaces and import the core. The core imports nothing from adapters.
2. **Infrastructure-owned ports** — Infrastructure defines the interfaces; domain code depends on them. This reverses dependency direction.
3. **Flat placement** — Domain logic, ports, and adapters share a single directory. Enforced only by discipline.

A core-owned layout:

```
core/
  orders/
    service.go
    order.go
    repository.go     ← port (interface owned by the domain)

adapters/
  postgres/
    order_repo.go     ← imports core/, implements the port
  redis/
  http/
```

**Trade-offs:**

[Strong Recommendation] **Core-owned ports** make the dependency rule mechanically enforceable. Because `postgres/` imports `core/`, a circular import in the reverse direction fails the build. The domain stays technology-independent. Swapping a database implementation requires changing only the adapter package. Infrastructure developers must understand the domain interfaces — a worthwhile cost.

**Infrastructure-owned ports** invert the coupling: the domain depends on infrastructure choices. Changing a database requires changing domain code. This is exactly the failure mode the pattern was designed to prevent.

**Flat placement** works at small scale with an unusually disciplined team. The architectural constraint lives in reviewers' heads. It erodes.

**Common failure modes:** A team adopts hexagonal architecture logically but colocates domain and adapters in a single package. An engineer imports an AWS SDK exception type directly into `PaymentService` because there is no directory boundary to stop them. The architecture degrades silently. The first observable symptom is often a database-specific exception appearing in a raw HTTP response.

**Example:** Rust enforces this pattern at the language level. A production Rust project defines a `core` module containing public traits (ports). A separate `infrastructure` module contains database adapters. Through explicit `pub use` re-exports, the `infrastructure` module exposes only selected public symbols, hiding concrete adapter implementations from the rest of the application. The module tree enforces information hiding; the compiler enforces dependency direction. No code review required.

---

### Circular Package Dependencies

**What it is:** The structural condition where package A imports package B and package B imports package A.

**Why it exists:** Circular dependencies are not naming problems. They are almost always a symptom of a concept split across a boundary it should never have crossed — two things that belong together have been forced apart, or a shared primitive has been left entangled in a higher-level feature package.

**Options:**

1. **Package merging** — Collapse the two mutually dependent packages into one.
2. **Primitive extraction** — Identify the shared type or interface causing the cycle and extract it into a third, independent package that both A and B import.

**Trade-offs:**

[Strong Recommendation] Choose **package merging** when the cycle is caused by overlapping domain concepts. If `orders/` imports `invoices/` to calculate a total, and `invoices/` imports `orders/` to read line items, the cycle reveals that these are one lifecycle domain, not two. The boundary was wrong. Merge them.

Choose **primitive extraction** when the cycle is caused by a shared mechanical type — a `RequestContext`, a common error type, a foundational event struct — that neither package truly owns. Extract it into a dedicated, dependency-free package.

**Common failure modes:** In languages that permit runtime circular imports (Python, Node.js), engineers mask the structural problem by moving the import statement inside the function body. The deferred import delays resolution until call time and hides the architectural failure. The result can be unpredictable runtime crashes when the two packages initialize each other's state in the wrong order under production load.

The second failure mode is extracting the shared type into `common/`, then adding more things to `common/` for convenience. The cycle disappears, but `common/` now imports everything and everything imports it — a god package wearing a different name.

**Example:** Go refuses to compile circular package imports. Engineers new to the language often find this restriction frustrating. It is a feature: the cycle is exposed at build time rather than discovered in a production incident. The structural boundary problem must be addressed rather than accumulated.

---

### The God Package

**What it is:** A package named `common/`, `shared/`, `helpers/`, or `utils/` that accumulates code whose only relationship is that no one knew where else to put it.

**Why it exists:** Creating a domain package requires making an ownership decision. Adding a function to `utils/` does not. Convenience replaces architecture one function at a time.

**Trade-offs:**

A `utils/` package typically grows to contain string helpers, date formatting, SQL generation, HTTP utilities, authentication helpers, configuration loading, and fragments of business logic that never found a proper home. None of these have anything to do with each other. Every other package imports `utils/` because it is convenient, making it the highest-coupled component in the system — high afferent coupling, near-zero internal cohesion.

The correct question is not "where should this go?" but "what *is* this?" Code that is genuinely shared deserves a package named after what it actually does. A date formatter and timezone parser belong in a `timeparse/` package. A money type and rounding rule belong in a `currency/` package. A structured logging adapter belongs in a `logging/` package. That name constrains what can accumulate there and makes ownership explicit.

[Strong Recommendation] A shared package is justified for genuinely generic, stable technical capabilities — logging, metrics, configuration, cryptographic primitives. The test: can the package be described in one noun phrase without listing its contents? If not, it is not an architectural layer; it is a drawer.

**Common failure modes:** A `utils/` package that starts with three string helpers grows to 3,000 lines over three years and becomes one of the top five most-imported packages in the repository. No team owns it, so every team modifies it. Attempts to split it require coordinating every team that imports it. It cannot be removed without a multi-quarter refactor, and it grows faster than it can be cleaned up.

**Example:** Large legacy Java codebases frequently develop a `common.jar` that is a required dependency of every other module. Its contents span serialization helpers, business rule fragments, database utilities, and domain types from a decade of "I'll just put it in common." No team owns it. Every team depends on it. A change to it requires a release of every downstream module, which is the reason changes to it rarely happen and the reason it keeps accumulating things.

---

### Why Smart Engineers Disagree on What Constitutes a "Feature"

The disagreement is not about whether package-by-feature is superior for application code. That question has largely settled. The disagreement is about what counts as a feature.

One engineer organizes around business domains: `orders/`, `billing/`, `inventory/`. Another organizes around team ownership boundaries: the concept is the same but the granularity is driven by Conway's Law rather than the domain model. A third engineer, building a query engine, uses `parsing/`, `planning/`, `execution/`, and `storage/` — all technical roles, but also the exact independent axes along which a query engine changes. For that engineer, the organizing principle *is* technical stage, and package-by-layer accurately reflects what evolves independently.

The correct organizing principle follows the primary axis of change. If independent business capabilities evolve at different rates, organize by business capability. If independent technical components evolve at different rates — because the product genuinely *is* a set of technical components — organize by technical component. The problem is not the scheme; it is choosing a scheme optimized for one axis when all actual changes happen along another.
