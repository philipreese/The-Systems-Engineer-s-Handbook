# Ch 46 — Spec-First Development

**Prerequisites:** [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md), [Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md), [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md), [Issue Tracking: What Makes a Good Issue](ch42-issue-tracking-what-makes-a-good-issue.md), [Architecture Decision Records (ADRs)](ch45-architecture-decision-records.md)

**New vocabulary introduced:** non-goal, spec theater, frozen spec

**Key takeaways:**
- A spec is a proposal for work not yet done, not documentation of work already done. Its entire purpose is to surface disagreement while changing direction is still cheap — before code, tests, and reviewer time have been sunk into a design nobody has actually agreed to yet.
- The economic argument is blunt: a flaw caught in a one-page proposal costs minutes to fix. The same flaw caught after implementation costs days of rework, and arrives at exactly the moment — PR review — when objecting to the underlying design is most expensive and most demoralizing.
- A spec describes interfaces, data flow, and constraints — not implementation. The moment a spec starts reading like pseudocode, reviewers debate naming and local structure instead of the architectural questions the spec exists to settle.
- An explicit *non-goal* — "this does not attempt X" — prevents scope creep more effectively than any review comment written after the fact, because it gives reviewers and stakeholders a boundary to point to instead of a boundary to infer.
- A spec's weight must match the decision's weight (Principle 6 applied directly): a one-way-door architectural change earns a full document and review cycle, a bounded feature earns a paragraph in the issue, a bug fix earns nothing. Mismatch either direction and you get *spec theater* or design review happening for the first time in a pull request.
- Once implementation begins, the spec has done its job. It is a proposal artifact, not living documentation — the *frozen spec*, quietly diverging from what was actually built, is a failure mode, not a maintenance backlog. What survives is the code, the tests, and the ADR ([Ch 45](ch45-architecture-decision-records.md)).

---

A specification describes work that hasn't been built yet. Its job is not to explain code to a future reader — that's what the code, the tests, and an ADR are for — but to expose a bad idea while it's still just a document, before it costs anyone real implementation time to discover the problem. This chapter argues for writing that document deliberately, and for matching its weight to how much the decision it's proposing actually matters. Recording a decision after it's made belongs to the ADR ([Ch 45](ch45-architecture-decision-records.md)); a spec's "alternatives considered" section often becomes the raw material for that later ADR, but this chapter is about the proposal, not the record. Where specs live long-term as a documentation type is covered in Part VIII.

---

### A Spec Moves Design Review to Where It's Cheapest

**What it is:** Writing and reviewing a proposal — the problem, the constraints, the proposed approach, the alternatives considered — before any implementation exists, instead of letting design disagreements surface for the first time during code review.

**Why it exists:** Most serious engineering mistakes are design mistakes, not implementation mistakes, and a design mistake is dramatically cheaper to fix in a document than in code. A reviewer can reject or reshape an idea in a document within minutes. The same disagreement, discovered after implementation, means unwinding days of work — and reviewers who recognize a better approach after weeks of implementation are, in practice, less willing to ask for the redesign it deserves. Late design review doesn't just cost more; it quietly becomes an evaluation of sunk cost rather than of engineering quality.

**Options:**
1. **Spec-first** — the design is proposed and reviewed as a document before implementation begins.
2. **Implementation-first** — the design is worked out through code, and review happens primarily at PR time.

**Trade-offs:**

[Strong Recommendation] **Spec-first, whenever the work represents a real design decision.** Implementation-first gets to visible progress immediately and skips any upfront documentation cost, but it defers the cheapest possible review to the most expensive possible moment: by the time a design flaw surfaces in a PR, there's already a body of code, tests, and reviewer expectations built around the flawed premise. Spec-first costs real time upfront and is unnecessary overhead for anything genuinely small, but for work that involves an actual design choice, it converts an expensive late discovery into a cheap early one.

**When to choose each:** Write the proposal first when the implementation represents a meaningful design decision rather than straightforward, already-understood execution. Skip it for routine bug fixes and small, well-understood changes — the proposal step only pays for itself when there's a real decision being reviewed.

**Common failure modes:** A reviewer spots a fundamentally flawed approach only after several thousand lines of code already implement it. The review technically succeeded — the flaw was caught — but it was caught at the worst possible point in the process, and the redesign now competes against the emotional and schedule cost of discarding real, finished-looking work.

**Example:** Amazon's "working backwards" process withholds engineering investment until a written narrative — the PR/FAQ — has been reviewed and agreed on, specifically to force the disagreement that would otherwise surface at launch to happen instead against a document nobody has built anything on top of yet.

---

### Describe Interfaces and Data Flow, Not Implementation

**What it is:** Restricting a spec to the problem statement, constraints, non-goals, proposed design at the level of interfaces and data flow, alternatives considered, and open questions — deliberately leaving class structure, algorithms, and line-level implementation to the implementation itself.

**Why it exists:** A spec's job is to let a reviewer evaluate whether the architecture is sound, not to pre-write the code. The moment a spec starts including concrete method signatures or pseudocode, review attention drifts toward naming and local structure — questions the implementation can answer far more cheaply than a document can — while the actual architectural risk, the thing the spec was written to catch, goes unexamined underneath all that detail.

**Options:**
1. **Interface-and-topology-first** — describe system behavior, data flow, boundaries, and constraints; leave internals to the implementer.
2. **Implementation-detail-heavy** — include concrete pseudocode, class hierarchies, and function signatures in the proposal itself.

**Trade-offs:**

[Strong Recommendation] **Interface-and-topology-first, as the default.** Implementation-heavy specs feel more concrete and give a junior engineer less ambiguity to resolve on execution day, but they duplicate work the implementation will do anyway, go stale the moment the implementer hits a real coding constraint the spec didn't anticipate, and flood reviewers with local detail that obscures the systemic question they're supposed to be answering. Interface-first specs demand more abstraction skill from both author and reviewer, but they keep review focused on what actually matters at this stage: is the shape of the system right.

**When to choose each:** Describe interfaces, responsibilities, and data flow; leave implementation details out unless a specific detail materially changes the architectural decision itself — a case where the low-level behavior of something like a custom concurrency primitive genuinely is the risk under review.

**Common failure modes:** A spec degenerates into pseudocode, and the review session spends most of its time debating variable names and local control flow before anyone has actually agreed whether the overall design is the right one to build at all.

**Example:** A well-formed Google design doc for a new geo-replicated configuration store specifies the service's request and response schemas, the maximum acceptable cross-region replication lag, and an explicit non-goal — no historical audit-log queries — while deliberately saying nothing about internal helper methods or buffer allocation strategy, keeping the review focused on replication behavior and consistency guarantees instead of code it hasn't been written yet to have opinions about.

---

### State Non-Goals as Explicitly as Goals

**What it is:** A dedicated statement of what the proposed work will *not* attempt, placed alongside the goals rather than left for a reader to infer.

**Why it exists:** Scope expands by default. Without an explicit boundary, every reviewer and stakeholder has room to suggest one more adjacent improvement, and the proposal drifts until it no longer represents a coherent, reviewable piece of work. A stated non-goal gives everyone something concrete to point back to instead of relitigating the boundary in every review comment.

**Options:**
1. **Explicit non-goals** — the spec states what it deliberately excludes.
2. **Implicit boundaries** — scope is left for the reader to infer from what the spec does say.

**Trade-offs:**

[Strong Recommendation] **Explicit non-goals in every substantial spec.** Implicit boundaries make for a shorter document, but they invite continual scope renegotiation — every reader arrives with a different assumption about what's in bounds, and each of those assumptions has to be corrected individually, usually more than once. Explicit non-goals cost one section of deliberate prioritization, in exchange for a proposal whose edges everyone agrees on before implementation starts.

**When to choose each:** Any spec substantial enough to warrant review at all should state its non-goals next to its goals — a reader should immediately see not just what the proposal accomplishes but what it's deliberately leaving for later.

**Common failure modes:** A spec proposing to expose internal inventory state to one reporting service never states what it excludes. During review, stakeholders quietly assume it will also handle multi-warehouse reservation logic, since nothing said otherwise — and the design expands under that unstated assumption until it's carrying scope nobody actually proposed or reviewed as a coherent whole.

**Example:** Design document templates that place "Goals" and "Non-Goals" as adjacent sections do so because the two together, not either alone, define what the proposal is actually asking to be reviewed on.

---

### Match the Spec's Weight to the Decision's Weight

**What it is:** Scaling how much a proposal costs to write and review — from a full design document down to a paragraph in the issue down to nothing at all — to how expensive the underlying decision would be to get wrong, per Principle 6: process only when it adds more than it costs.

**Why it exists:** A spec is process, and like any process it has to justify its cost against the risk it's managing. A one-way-door architectural change — a new persistence model, a change to multi-tenant routing — has enormous blast radius and deserves the full review cycle. A bounded feature addition is comparatively cheap to get wrong and cheap to fix, and a short paragraph of design intent inside the issue is enough. A routine bug fix carries essentially no architectural risk and doesn't need a proposal at all.

**Options:**
1. **Heavyweight design document** — a full written proposal with formal, often cross-team review.
2. **Lightweight proposal** — a brief design captured directly in the issue.
3. **No specification** — implementation proceeds directly.

**Trade-offs:**

A heavyweight document buys thorough, wide-visibility review appropriate for a decision that's genuinely hard to reverse, at the cost of real time and reviewer attention that would be wasted on anything smaller. A lightweight proposal is fast and sufficient for bounded, easily-reversible work, but doesn't provide the depth of exploration a major architectural change actually needs. Skipping the spec entirely is fastest of all, and correct for routine work — but applied to something with real architectural weight, it just relocates design review to the PR, where changing course is far more expensive.

**When to choose each:** Apply [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md)'s reversibility-and-blast-radius framework directly: hard-to-reverse, wide-blast-radius decisions get the full document; bounded feature work usually needs only a short proposal inside the issue itself; routine fixes need none.

**Common failure modes:** *Spec theater.* A team requires a full design document for every change, including trivial ones, and the documents get written — after the implementation is already finished — purely to satisfy the checklist. The review happens, but it happens against a document describing a decision that was made and shipped before anyone read it, which means the review changed nothing; it only produced paperwork. The opposite failure is a team that writes no specs at all, so architecture review migrates entirely into pull requests, where it's most expensive and least welcome.

**Example:** The IETF's RFC process, Python's PEPs, and Rust's RFC process all deliberately impose substantial review overhead, because a language-level change is both extremely hard to reverse and affects an enormous number of downstream users — the same review weight applied to an ordinary internal feature would impose a cost far beyond the risk it's managing.

---

### A Spec Is a Proposal Artifact, Not Living Documentation

**What it is:** Treating a spec as a record of the proposal that guided implementation — frozen once implementation begins — rather than a document that's kept continuously synchronized with the system as it evolves.

**Why it exists:** Implementations evolve in ways specs don't predict: patches, scaling fixes, and hotfixes accumulate long after the original proposal was reviewed. A spec that tries to stay "current" forever either demands a maintenance discipline nobody actually sustains under real deadline pressure, or quietly falls out of sync while still looking authoritative — which is worse than having no spec at all, because a reader trusts it without knowing it's wrong.

**Options:**
1. **Proposal artifact** — the spec is archived once implementation begins; the code, tests, and any resulting ADR become the authoritative sources going forward.
2. **Living document** — the spec is continuously updated to track the system's current behavior.

**Trade-offs:**

[Strong Recommendation] **Treat specs as proposal artifacts, archived once their purpose is served.** A living spec sounds appealing — one authoritative prose description of the whole system — but it demands a synchronization discipline that reliably breaks down under hotfix pressure and shipping deadlines, and a spec that's supposed to be current but isn't is more dangerous than an openly historical one, because nothing signals to the reader that it can no longer be trusted. An archived proposal costs nothing to maintain and stays historically accurate about what was proposed and why — it just stops being the source of truth for current behavior, which the code, tests, and ADR trail were always better positioned to be anyway.

**When to choose each:** Archive the spec once implementation is underway. Reserve continuous synchronization for narrow cases where documentation is generated directly from source — an API reference built from code annotations — rather than maintained by hand as separate prose.

**Common failure modes:** *The frozen spec.* A new engineer finds a detailed, well-written design document describing a service's persistence layer as synchronous writes to a single database. The team migrated that layer to an asynchronous, eventually-consistent pipeline over a year ago and never updated the document. The engineer builds an optimization on the documented — and no longer true — assumption, introducing a data-loss risk the actual running system had already designed around.

**Example:** Teams that treat specs as historical artifacts mark a completed design document with a note pointing forward — "for the current architecture, consult the ADR directory and the active test suite" — rather than leaving it looking like a live reference. The rule this preserves: what survives past implementation is the code, the tests, and the ADR chain describing what was actually decided ([Ch 45](ch45-architecture-decision-records.md)), not the original proposal's prose.

---

### Commit Once Enough Is Known

**What it is:** Treating the spec review cycle as something with a natural stopping point — implementation begins once the major uncertainties are resolved, rather than continuing indefinitely in pursuit of additional confidence.

**Why it exists:** Analysis has diminishing returns. Past the point where the significant risks in a proposal have been surfaced and addressed, further review cycles mostly relitigate settled questions rather than catching new ones. [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md) already established that reversible decisions deserve speed, not extended study — a spec cycle that keeps running past that point isn't reducing risk anymore, it's substituting for the decision itself.

**Options:**
1. **Decision-oriented review** — review until confidence is sufficient, then implement.
2. **Open-ended review** — continue refining the proposal without ever committing to it.

**Trade-offs:**

[Strong Recommendation] **Decision-oriented review, calibrated to the decision's actual reversibility.** Open-ended review can genuinely surface additional alternatives and raise confidence, but past a certain point it stops producing new information and just delays execution — and for a reversible decision, that delay is pure cost with no corresponding risk reduction to show for it. Decision-oriented review accepts that some uncertainty will remain when implementation starts, which is the correct trade for anything that can be corrected cheaply later.

**When to choose each:** For reversible decisions, move to implementation once the proposal is good enough — perfect confidence isn't the bar. Reserve genuinely extended review cycles for decisions that are actually expensive to reverse.

**Common failure modes:** *Analysis paralysis.* A spec for a low-risk, easily-reversible internal change enters a review cycle that drags on for months, accumulating comments long after the substantive questions were already answered. The spec process has stopped being a review mechanism and started being a way to avoid committing to a decision at all.

**Example:** Language-level RFC processes move deliberately slowly because the changes they govern are extremely hard to reverse and affect a huge population of users — a justified cost there that would be actively harmful if applied to an ordinary, easily-reversible application feature.

---

### Why Smart Engineers Disagree on the Discovery Medium

A deeper disagreement than "how much spec" is whether writing text before code is even the right way to discover a design's real constraints in the first place.

One position holds that clear writing is the closest available proxy for clear thinking: forcing an engineer to state a system's data invariants and defend its alternatives in prose exposes hand-waving and unhandled edge cases that thousands of lines of code can otherwise hide underneath the appearance of progress. To this view, code-first development risks burying genuine design ambiguity under implementation activity that looks like progress but hasn't actually resolved anything. The opposing position argues that a text document is a passive medium that can't push back: an engineer can write an internally consistent spec for an integration and only discover on the first day of actually coding it that the external system has undocumented thread-safety limits or blocking behavior that invalidates the design entirely — something no amount of careful prose would have surfaced, because the constraint lived in a dependency's actual runtime behavior, not in anyone's stated assumptions.

Both positions are right about where their preferred medium's real risk sits. When a design's biggest uncertainty is organizational agreement, service boundaries, or business logic that lives entirely in stated intent, a written spec is close to unbeatable at surfacing disagreement cheaply. When the biggest uncertainty is how an unfamiliar runtime, third-party library, or piece of hardware actually behaves under real conditions, a spec can only formalize an assumption — it can't test one. The practical answer isn't to pick a single medium for every proposal: run a deliberately time-boxed prototype spike first when the risk is genuinely in unproven runtime behavior, then require that spike's conclusions be written up as an interface-level spec — reviewed the way any other proposal would be — before it hardens into production infrastructure that other teams have to build against.
