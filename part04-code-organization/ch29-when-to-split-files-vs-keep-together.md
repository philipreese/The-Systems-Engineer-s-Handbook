# Ch 29 — When to Split Files vs. Keep Together

**Prerequisites:** [Coupling and Cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md), [Abstraction and Information Hiding](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md) (specifically: the Rule of Three), [File and Module Structure](ch27-file-and-module-structure.md)

**New vocabulary introduced:** None beyond concepts established in prior chapters. This chapter applies [cohesion](../part01-systems-thinking/ch03-coupling-and-cohesion.md) and the [Rule of Three](../part01-systems-thinking/ch04-abstraction-and-information-hiding.md) at the smallest practical grain.

**Key takeaways:**
- A file is a unit of comprehension, not a unit of syntax. The question is not how many declarations fit, but whether they belong together.
- Split on falling cohesion, not rising line count. A thousand-line parser implementing one algorithm can be easier to read than a two-hundred-line file mixing authentication, logging, and business rules.
- The signal to split is that readers naturally think "this part is about something different." The signal to keep together is that the pieces lose meaning apart.
- A state machine whose transitions are scattered across six files is harder to understand than one that lives in a single cohesive file — even a long one.
- Test and asset co-location follows the same cohesion reasoning: keep with the source what is understood and changed alongside it.

---

[Ch 27](ch27-file-and-module-structure.md) established package- and module-level organization. This chapter assumes that boundary already exists and asks a finer-grained question: within a module, should this concept live in its own file or alongside closely related code? Whether splitting a file introduces a new abstraction boundary worth its cost is a separate question, covered in [Ch 31](ch31-when-abstractions-help-vs-when-they-obscure.md).

---

### One Entity Per File or One Concept Per File?

**What it is:** The baseline convention for how source code is distributed across files — whether a file is an exact index of individual types, or a container for one cohesive concept that may span several closely related types and functions.

**Why it exists:** Files exist to help humans navigate a codebase. The convention determines what the file system indexes: individual named entities, or holistic ideas.

**Options:**

1. **One class per file** — each type, struct, or interface gets its own dedicated file, named after the entity it contains.
2. **One concept per file** — a file contains whatever declarations are needed to understand one cohesive idea: the primary type, its small supporting types, helper functions, and internal constants.

**Trade-offs:**

**One class per file** makes finding any specific entity trivial via file-tree navigation — you know exactly which file it's in. The cost is reading flow. A small helper type that is exclusively consumed by one parent class has no independent identity, but isolation forces the reader to open a second file to understand a single operation. Java's enforcement of this convention — one public class per file — produced directories cluttered with `UserActionFactory.java`, `UserActionBuilder.java`, `UserActionBuilderFactory.java`: dozens of files for simple workflows, a high navigation tax on shallow logic.

[Strong Recommendation] **One concept per file** is the dominant convention in Go, Python, and JavaScript/TypeScript, and for good reason. It preserves reading flow: a developer can understand the entire lifecycle of a domain concept by scrolling rather than tab-switching. An interface, the struct that implements it, and the small helper types it relies on naturally share one file. The risk is the god file — the canonical failure of this approach, where the "concept" boundary softens and unrelated logic accumulates indefinitely.

**When to choose each:** Follow the idiomatic convention for the language. Java enforces one public class per file at the language level; respect it. In Go, Python, and JavaScript, prefer conceptual co-location for small, tightly related declarations that have no distinct identity outside their immediate context. Large public types with independent public APIs earn their own files in any language.

**Common failure modes:** Mechanically creating a new file for every enum, exception, helper type, or small interface. A feature that would take ten minutes to understand now requires opening fifteen files. The organizational rule has increased navigation cost without clarifying anything. The opposite failure is equally common: unrelated types accumulate indefinitely because "they already happen to be in the same file."

---

### Split on Falling Cohesion, Not Rising Line Count

**What it is:** The signal that a file should be split is that it contains multiple independent concerns — not that it exceeds an arbitrary line threshold.

**Why it exists:** Length alone rarely makes code difficult. Unrelated ideas do. A thousand-line parser implementing one cohesive algorithm is easier to read than a two-hundred-line file that mixes authentication, logging, configuration, and business rules. The difficulty in the second case is not the size — it's that no single organizing concept holds the pieces together.

**Options:**

1. **Responsibility-driven splitting** — split when a file visibly contains multiple concerns with low cohesion between them.
2. **Threshold-based splitting** — split when the file crosses a line count or declaration count limit.
3. **Keep together** — maintain cohesive code in one file regardless of growth.

**Trade-offs:**

[Strong Recommendation] **Responsibility-driven splitting** reflects what actually makes files hard to maintain. The Rule of Three applies here exactly as it does to abstraction: wait until multiple independent responsibilities have clearly emerged before introducing the organizational overhead of a new file. Splitting prematurely predicts an evolution that often never comes, leaving dozens of ten-line files that each contain too little context to stand alone.

**Threshold-based splitting** is easy to automate and enforce in code review, which explains its appeal. It is wrong. It treats line count as a proxy for cohesion, and the proxy fails in both directions: a long cohesive file gets broken up unnecessarily, while a short incoherent one passes the check.

**When to split:** When modifying one responsibility in the file consistently requires ignoring the rest. When a reader has to use text search to navigate between two related functions separated by hundreds of lines of unrelated logic. When the file contains code that changes at visibly different rates for visibly different reasons.

**When to keep together:** When the code shares a single axis of variation. When splitting would require a reader to trace execution across multiple files to understand one operation.

**Common failure modes:** A growing file reaches 500 lines. Rather than identifying the conceptual boundary, a developer divides it into `billing_part1.py`, `billing_part2.py`, and `billing_helpers.py`. The original cohesion problem is unchanged; the navigation problem is now worse. The split communicated no information about what actually changed.

**Example:** An `auth_middleware.go` file that intercepts an HTTP request, validates a JWT, and returns a `401 Unauthorized` should stay in one file — those three steps are one cohesive operation. If that same file accumulates a custom base64 decoding algorithm and an RSA key-rotation cron job, those belong in `jwt_decoder.go` and `key_rotation.go`. They are independent axes of variation: they change for different reasons, at different times, in response to different requirements.

---

### Keep Tightly Coupled Pieces Together

**What it is:** Some code loses meaning when separated. Small mutually dependent types, finite state machines, and tightly coordinated internal logic are often harder to understand distributed across files than collected in one.

**Why it exists:** Navigation has a cost. Every additional file is another place a reader must search. When two pieces of code are almost always read together — because understanding one requires knowing the other — separation adds indirection without adding clarity.

**Signals that pieces belong together:**
- Understanding a state transition requires reading the state definition, the event types, and the transition logic simultaneously
- A small type has no meaningful identity outside the one file that uses it
- Splitting would require importing from sibling files within the same module rather than from the module interface

**Common failure modes:** An engineer applies a 100-line file limit dogmatically. A cohesive 300-line state machine is shattered across six files. Understanding a single transition now requires opening six tabs and tracing imports back and forth. The fragmented state machine has higher afferent and efferent coupling at the file level than the original monolithic one did — the opposite of the intended outcome.

**Example:** Many parser implementations keep token definitions, parser state, and transition logic in one file because the three concepts are inseparable during maintenance. Reading a grammar rule requires seeing the token type, the parser state it appears in, and the transition it triggers. Distributing these across files forces the reader to reconstruct mentally what the file structure once made visually available.

---

### Test and Asset Co-Location

The same cohesion reasoning that governs source file organization applies to tests and related assets. A test file exists to verify the contract of one source file; a stylesheet or template often exists to define the visual behavior of one component. The question of where they live is the same question as where closely related source types live.

Two conventions exist:

1. **Adjacent co-location** — the test and source live in the same directory: `billing.go` and `billing_test.go`, `Button.tsx` and `Button.test.tsx`.
2. **Parallel tree** — a separate root directory mirrors the source structure: `src/billing/service.java` and `test/billing/ServiceTest.java`.

Modern frontend frameworks have largely converged on adjacent co-location as a deliberate cohesion-first choice. A React component folder containing `Button.tsx`, `Button.css`, and `Button.test.tsx` keeps the full concept of a button in one place. The older layered convention — all components in `components/`, all styles in `styles/`, all tests in `tests/` — mirrors package-by-layer: it organizes by artifact type rather than by the concept that changes together.

The primary failure mode of parallel trees is the orphaned test: a developer refactors `src/` and renames several files but forgets to execute the same structural changes in `test/`. The tests become disjointed from the code they verify, and the mismatch isn't discovered until a test is needed.

Testing strategy itself is covered in Part V. The placement decision here is a cohesion question, not a test design question, and the answer is the same: keep together the things that are understood and changed together.

---

### Why Smart Engineers Disagree: Scrolling vs. Tab-Switching

The file granularity debate is usually framed as a preference argument. It isn't. It's a disagreement about which cognitive tax scales worse.

Engineers who prefer small files argue that any file over 200 lines is a smell. They claim small files enforce the Single Responsibility Principle and that scrolling is a sign of an overgrown monolith. Engineers who prefer larger files argue that tab-switching shatters context. Every time a developer opens a new file to find a helper function definition, they risk losing their place in the operation they were tracing.

Both taxes are real. But they scale differently. The cost of scrolling through a cohesive file is roughly linear: more content, more scrolling. The cost of tab-switching is closer to exponential: each new file is a context-switch that risks a cache miss in the developer's working memory. The execution path has to be reconstructed from scratch on each switch.

The deciding factor is never the raw line count — it is the density of cohesion. A 1,500-line file implementing one cohesive protocol parser is harder to navigate than it appears in summary and easier to understand than ten 150-line files would be. If scrolling through a file feels like searching a junk drawer — if the next section is always about something different — the file lacks cohesion and should be split. If scrolling feels like following a narrative, the file is working correctly, regardless of length.
