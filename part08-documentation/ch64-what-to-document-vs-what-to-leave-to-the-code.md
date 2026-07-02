# Ch 64 — What to Document vs. What to Leave to the Code

**Prerequisites:** [Complexity Is the Enemy](../part01-systems-thinking/ch02-complexity-is-the-enemy.md), Principle 6, [File and Module Structure](../part04-code-organization/ch27-file-and-module-structure.md), [Naming Conventions and When They Matter](../part04-code-organization/ch28-naming-conventions-and-when-they-matter.md), [Comments: What to Comment, What Not To](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md)

**New vocabulary introduced:** documentation tax

**Key takeaways:**
- External documentation is not the default response to confusing code — it is what's left after a code-level fix (better naming, clearer structure, a WHY comment) has been ruled out. Reach for it only when information has no single location a comment could occupy.
- Every page of external documentation carries a documentation tax: an ongoing cost of being located, read, trusted, and eventually corrected, paid whether or not the page turns out to have been worth writing.
- Information that clears the bar for external documentation falls into a small number of recurring categories — onboarding context, decision rationale, operational knowledge — that answer different questions for different readers and should not be collapsed into one undifferentiated document.
- Documentation coverage is a proxy metric, and Principle 9 applies to it exactly as it applies to test coverage (Ch 41): a mandated docstring percentage gets satisfied by filler prose that repeats the function's name back to the reader, not by anything worth knowing.
- The right governance mechanism is review scrutiny, not a coverage gate — a reviewer can reject a page for adding no information; a linter can only confirm one exists.

---

This chapter opens Part VIII by extending [Ch 30](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md)'s WHY-vs-WHAT test one level up. That chapter asked whether a specific comment, attached to a specific line, earns its keep. This chapter asks the prior question: whether any external documentation is warranted at all, before which artifact it becomes (Ch 65) is even a live question. Ch 02's complexity argument and Principle 6 apply unchanged — documentation is a cost paid to reduce complexity for a future reader, not a free good, and it should exist only when its value exceeds that cost.

### Decision: Fix the Code Before You Document It

**What it is:** The default test applied to any information gap — can this be resolved by improving the code itself, or does it genuinely have nowhere else to live?

**Why it exists:** Most unwarranted documentation exists because writing a paragraph feels cheaper than refactoring. It usually isn't. A code-level fix is enforced by the same review and the same compiler that touch the code around it; a document is not. If the code can communicate the information, external documentation duplicates it and starts drifting from the moment it's published.

**Options:**
1. **Fix the code** — better names ([Ch 28](../part04-code-organization/ch28-naming-conventions-and-when-they-matter.md)), clearer module boundaries ([Ch 27](../part04-code-organization/ch27-file-and-module-structure.md)), a stronger type, or a localized WHY comment ([Ch 30](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md)).
2. **Write external documentation** — only when the information has no reasonable location inside the codebase.

**Trade-offs:**

| | Code-level fix | External documentation |
|---|---|---|
| Currency | Automatically current — it *is* the implementation | Pays a documentation tax indefinitely; can go wrong while the code it describes stays right |
| Discoverability | Found by reading the code that needs it | Must be located, then trusted, before it's useful |
| Expressive range | Cannot capture information with no single location — rejected alternatives, cross-service intent, anything true before a reader has opened any file | Can describe whole-system behavior and orient a reader who hasn't read anything yet |

[Strong Recommendation] Default to the code-level fix. Reach for external documentation only when the fix genuinely cannot carry the information — not because refactoring is more work right now than writing a paragraph.

**When to choose each:** External documentation is justified only when information spans multiple files, multiple services, or multiple points in time that no single comment location could cover. It is not justified because the code is currently confusing — that is a refactor, not a documentation gap.

**Common failure modes:** *The textual patch.* An engineer meets a tangled, over-coupled module ([Ch 03](../part01-systems-thinking/ch03-coupling-and-cohesion.md) territory) and writes a wiki page explaining how to navigate it safely, instead of fixing the coupling. The document now has to be maintained forever, and it goes stale the first time someone touches the module without reading the wiki first — leaving both the bad code and a wrong map of it in place at once.

**Example:** Kubernetes' control-plane responsibilities, reconciliation model, and extension mechanisms cannot be reconstructed from source comments alone — they span thousands of files across dozens of independently evolving components, which is exactly the no-single-location condition that justifies documentation living outside any of them.

---

### Decision: Name What Has Nowhere Else to Live

**What it is:** The recurring categories of information that clear the bar set above, independent of which specific artifact (Ch 65) each eventually becomes.

**Why it exists:** Code is local — a file, a function, a module. Some information isn't: it's true across an entire service, true before a reader has opened any file at all, or relevant only during a five-minute window at 3 a.m. No single comment location can hold information like that.

**Options** (the recurring categories):
1. **Onboarding context** — what a newcomer needs before they can productively read the first source file: repository purpose, build prerequisites, where to start. A README's job, taxonomy resolved in Ch 65.
2. **Decision rationale** — a permanent record of a decision and the alternatives rejected, preserved past the point the current code alone could reconstruct why. An ADR's job ([Ch 45](../part06-engineering-process/ch45-architecture-decision-records.md), referenced here, not re-derived).
3. **Operational knowledge** — facts that matter only while a system is actively failing: what to check, what to run, who to escalate to. A runbook's job (Ch 68).

**Trade-offs:** These three don't compete — they answer different questions for different readers at different times. The risk is conflating them: folding decision rationale into a README turns a fixed historical record into something a reader expects to be current, which Ch 65 identifies as the single most common taxonomy mistake; burying operational steps inside general architecture prose forces a responder mid-incident to read material written for a different reader entirely.

**When to choose each:** Onboarding context whenever a reader's first need is orientation, not implementation detail. Decision rationale whenever the reasoning behind a rejected alternative would otherwise be lost once the person who made the call moves on. Operational knowledge whenever the reader is responding to an incident, not doing ordinary development work.

**Common failure modes:** An architectural rationale, written to justify one decision at one point in time, gets pasted into a README because no ADR directory exists yet. It now reads as current guidance long after the trade-off it describes has stopped applying, and nothing marks it as historical.

**Example:** Google's internal engineering practices documentation draws exactly this line explicitly — code comments are restricted to what the code cannot say for itself, while onboarding guides, design rationale, and operational documentation are deliberately maintained as separate, differently-scoped artifacts rather than one undifferentiated wiki.

---

### Decision: Govern Documentation by Scrutiny, Not by Coverage

**What it is:** Whether documentation quality is enforced by an automated completeness metric — docstring coverage, comment coverage — or by review judgment applied to whether each page earns its documentation tax.

**Why it exists:** This is Principle 9 applied to prose instead of tests. The moment "percentage of functions with a docstring" becomes a CI gate, engineers satisfy the gate, not the reader. [Ch 41](../part05-testing-strategy/ch41-coverage-what-it-measures-and-what-it-doesnt.md) already showed this failure for test coverage; documentation coverage fails identically and for the same reason — a percentage measures that something was written, never whether it was worth writing or still true.

**Options:**
1. **Metric-driven coverage enforcement** — a linter blocks merges below some docstring or comment coverage percentage.
2. **Review-driven scrutiny** — documentation changes go through the same pull request review as code, and a reviewer can reject a page for repeating what the code already says, not merely approve it for technically existing.

**Trade-offs:**

| | Metric-driven coverage | Review-driven scrutiny |
|---|---|---|
| Guarantees | Something exists everywhere; satisfies external audit and tooling requirements | Nothing automated; depends entirely on reviewer discipline |
| Failure mode | Engineers write filler to pass the linter | None if reviewers actually apply the WHY test — costs reviewer attention |
| What survives | High volume, low signal | Low volume, high signal |

[Strong Recommendation] Review-driven scrutiny for internal documentation — application code, internal services, infrastructure. Reserve metric-driven coverage for the one case where the metric approximates something real: a published SDK or external API surface (Ch 67), where the coverage tool is the same tool generating the reference a consumer will actually read.

**Common failure modes:** *The tautological linter stub.* A CI rule requires a docstring on every public function. An engineer satisfies it with prose that adds nothing beyond the signature:

| Code signature | Stub docstring |
|---|---|
| `func FetchUserByID(id string) (*User, error)` | `// FetchUserByID fetches a user by their ID.` |
| `func IsCacheValid(ts int64) bool` | `// IsCacheValid checks if the cache is valid based on timestamp.` |

The metric improves. The documentation does not — and it will drift the moment either function's behavior changes, since nothing forces the stub to change with it.

**Example:** The Docs as Code movement (Write the Docs) treats documentation changes as pull requests subject to the same review as code: a reviewer who finds a page duplicating information already visible from a clean type signature or a well-named function rejects the change, rather than merging it because a coverage check passed.

---

### Why Smart Engineers Disagree on Documentation Volume

One position — call it the code-purist view — treats almost all external documentation beyond a bare-minimum architecture overview as a smell: if a system needs a document to explain how it works, the design has failed Ch 02's complexity test, and the fix is better names, decoupled modules, or a precise inline comment. Prose is an uncompiled, untested liability that starts rotting the moment it's merged.

The opposing position — call it the context-pragmatist view — argues that even perfectly clean code is structurally blind to what's absent from it: why an obvious approach was rejected, what real-world constraint forced a counterintuitive choice, how a business pivot three years ago left an imprint on the schema. No amount of naming discipline recovers information the code never contained in the first place.

Both positions agree documentation has a real cost; they disagree about how often that cost is worth paying, and the disagreement tracks organizational scale more than either side usually credits. A stable team maintaining a compact codebase has continuous shared context — the code purist's environment, where the documentation tax rarely pays for itself. A large organization with distributed ownership and real turnover loses that shared context constantly — the context pragmatist's environment, where reconstructing historical intent from code alone becomes a genuine bottleneck to cross-team work. Neither position argues for documenting what a well-named function or type already says; they differ only on how much information a given system actually has that code cannot express.
