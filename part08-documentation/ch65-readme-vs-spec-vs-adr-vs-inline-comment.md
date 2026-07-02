# Ch 65 — README vs. Spec vs. ADR vs. Inline Comment

**Prerequisites:** [Comments: What to Comment, What Not To](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md), [Architecture Decision Records (ADRs)](../part06-engineering-process/ch45-architecture-decision-records.md), [Spec-First Development](../part06-engineering-process/ch46-spec-first-development.md), [What to Document vs. What to Leave to the Code](ch64-what-to-document-vs-what-to-leave-to-the-code.md)

**New vocabulary introduced:** living document

**Key takeaways:**
- Every documentation artifact is placed by two independent questions: who is the intended reader, and does the artifact need to track current reality or is it allowed to be a fixed record of a past moment.
- A README is a living document: it must always describe the repository as it exists today. A stale README is a documentation failure, because it costs a newcomer their first, most trust-forming hour.
- A spec and an ADR are point-in-time records: staleness is expected, not a failure, provided both are treated as archived artifacts once implementation begins rather than living references.
- Conflating those two categories — editing an ADR to match today's architecture, or trusting an old spec as a current reference — is the single most common taxonomy mistake, and the opposite failure from a stale README.
- The selection test is: identify who is asking, what question they're asking, and whether the answer needs to still be true tomorrow. That third question alone routes almost every case correctly.

---

Four documentation artifacts have already appeared across this handbook: inline comments ([Ch 30](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md)), ADRs ([Ch 45](../part06-engineering-process/ch45-architecture-decision-records.md)), specs ([Ch 46](../part06-engineering-process/ch46-spec-first-development.md)), and READMEs (named in [Ch 64](ch64-what-to-document-vs-what-to-leave-to-the-code.md)'s onboarding-context category). This chapter places them side by side and resolves which one a given piece of information becomes. It does not re-argue why specs and ADRs exist, or when an inline comment earns its cost — those cases were already made. It answers a narrower, prior question: given that something is worth writing down externally (Ch 64), which of the four does it become?

### Decision: Two Axes Place Every Artifact

**What it is:** A classification of documentation artifacts along two independent dimensions — audience and lifespan — that together determine which artifact a piece of information belongs in.

**Why it exists:** Without an explicit test, documentation placement defaults to habit: architectural context gets buried in whichever file the author happened to be editing, and README files accumulate design history that was never meant to stay current. The two axes make the choice mechanical instead of arbitrary.

**Options:** The two questions to ask of any candidate artifact:
1. **Audience** — is the reader someone approaching the repository for the first time, someone reading one function, someone planning future work, or someone trying to understand a decision already made?
2. **Lifespan** — must the artifact continue to describe the system as it exists today, or is it allowed to permanently preserve what was true at one point in time?

**Trade-offs:**

| Artifact | Primary reader | Tracks current reality? | Primary purpose |
|---|---|---|---|
| Inline comment | Someone reading the surrounding code | Yes, locally | Explain intent tied to specific lines |
| README | Someone approaching the repository | Yes | Orient and onboard |
| Specification | Engineers evaluating proposed work | No — archived once implementation begins | Describe work not yet done |
| ADR | Someone revisiting a past decision | No — by design | Record why a decision was made and what was rejected |

**When to choose each:** The lifespan axis does the heavier lifting. If the information must still be true next quarter for the artifact to be useful, it belongs in a living document. If the information is valuable specifically *because* it freezes a moment — a proposal, a decision and its rejected alternatives — forcing it to track current reality destroys the thing that made it worth writing.

**Common failure modes:** Treating the two axes as one axis — assuming anything "important enough to write down" should also be kept current, which is exactly the assumption that turns ADRs into architecture documentation and specs into reference manuals nobody trusts.

**Example:** A newcomer opening a repository is asking "what is this and how do I run it" — a README question. An engineer investigating a contested architectural choice is asking "why did we choose this over the alternatives" — an ADR question. Both are legitimate, both are common, and neither artifact can answer the other's question well.

---

### Decision: A README Is a Living Document, Not an Archive

**What it is:** The repository's entry point for a reader who has not yet understood the code — what this is, how to build and run it, and where to go next.

**Why it exists:** A reader needs orientation before implementation detail is meaningful, and that orientation has to be trustworthy on first contact, since a newcomer has no other basis yet for judging whether it's accurate.

**Options:**
1. **Keep the README focused** on orientation, pointing to specs, ADRs, and deeper docs rather than absorbing their content.
2. **Let the README expand indefinitely** into the repository's single documentation surface.

**Trade-offs:** A focused README is easy to keep current and easy for a newcomer to finish reading, at the cost of not containing every detail — but it was never supposed to. A README that expands to absorb design history and proposal discussions becomes exactly the kind of point-in-time content the next decision covers, except now it's mixed into an artifact that's supposed to always be current — so parts of it silently stop being true and nothing signals which parts.

[Strong Recommendation] A README that is always roughly current beats a README that is comprehensive. Cut anything that isn't true today into its own artifact rather than let it dilute the one document a newcomer is trusting by default.

**When to choose each:** A README whenever the reader is approaching the repository for the first time and the answer needs to be true today, not whenever information feels important enough to be visible.

**Common failure modes:** Installation instructions that no longer match the build system. References to directories that were renamed two refactors ago. A README that reads as an archive of every design discussion the project ever had, none of which a newcomer asked for.

**Example:** GitHub renders a repository's README automatically on its landing page — a convention strong enough that it has made the README the de facto front door for software projects. New contributors encounter it before any source file, which is exactly why its accuracy is not optional the way a design document's is.

---

### Decision: Specs and ADRs Are Point-in-Time Records, Not Living References

**What it is:** Both a specification and an ADR are permanent records of a specific moment — a proposal not yet built, or a decision already made — rather than descriptions of the system as it exists now.

**Why it exists:** A spec exists to coordinate implementation before implementation exists; once implementation begins, it becomes a historical proposal, and expecting it to keep evolving alongside the code duplicates the code as documentation and inevitably falls behind it. An ADR exists to record why a decision won over its rejected alternatives; that reasoning stays true forever even after the system built on it changes, and rewriting the ADR to match today's architecture destroys the exact thing it was written to preserve.

**Options:**
1. **Preserve as an archived snapshot** — leave the spec or ADR exactly as it was at the moment implementation began or the decision was made.
2. **Continuously update it to match current reality** — edit the original document as the system evolves.

**Trade-offs:** Preserving the snapshot keeps the original reasoning intact — why an alternative was rejected, what was actually proposed — at the cost of the document no longer describing the running system. Continuously updating it can look more current, but it erases the historical record and creates a second, unofficial description of the system that has to be kept in sync with the code by hand, with no compiler or reviewer forcing that sync the way there is for code itself.

[Consensus] Once implementation begins, a spec is archived, not maintained. Once a decision is made, an ADR is left as-is; a later change in direction gets a *new* ADR that references and supersedes it, not an edit to the original. This is the single most common taxonomy mistake this chapter corrects: a stale spec or ADR is not the same failure as a stale README — it's the artifact doing exactly what it was for.

**Common failure modes:** An engineer treats a two-year-old spec as an authoritative description of the current system and builds a new feature on assumptions the implementation abandoned months ago. In the opposite direction, a team edits a three-year-old ADR to reflect a database migration that happened last quarter — the ADR now claims the original team chose the *new* database, erasing the actual constraint that justified the original, now-superseded choice.

**Example:** The Kubernetes Enhancement Proposal (KEP) process records the motivation, design, and rejected alternatives for a proposed change, entirely separate from Kubernetes' user-facing reference documentation, which describes the system as it actually ships. Nygard-style ADR directories follow the same discipline: a project's `/adr` folder sits alongside its README, and the two are never expected to agree with each other in the way a README and the code it describes are — the README explains what exists now, the ADR directory explains why yesterday's decisions were made, and confusing the two is the error, not a symptom of either being wrong.

---

### Decision: Ask the Question Before Picking the Artifact

**What it is:** The practical selection test that operationalizes the two axes above — identify the reader and the question before deciding where an answer belongs.

**Why it exists:** "Where should I write this?" is the wrong first question; it invites habit and convenience over correctness. "What question is someone trying to answer, and does the answer need to still be true tomorrow?" routes almost every case correctly on its own.

**Options:** Apply the test to the specific question at hand:
- *"What do I need to run this repository right now?"* → **README.** The answer must be true today.
- *"What are we proposing to build, and how will we validate it?"* → **Spec.** The answer is a snapshot of a plan, archived once building starts.
- *"Why did we choose this approach over the alternatives?"* → **ADR.** The answer is a snapshot of a decision, frozen at the moment it was made.
- *"Why does this specific line do something counterintuitive?"* → **Inline comment.** The answer is tied to that line and moves or dies with it.

**Trade-offs:** Choosing by audience alone improves discoverability but doesn't resolve maintenance expectations; choosing by lifespan alone resolves maintenance expectations but doesn't tell a reader where to look first. The test needs both — audience narrows the candidates, lifespan picks between what's left.

**When to choose each:** Whenever a decision about where to record something is being made, ask both questions before writing anything. If the artifact already exists and the question is "is this still in the right place," the same test still applies retroactively.

**Common failure modes:** A team skips the "must this stay current" question and writes a spec as if it were reference documentation, then treats every implementation deviation from the spec as a bug in the spec rather than an expected divergence.

**Example:** Two engineers ask different questions about the same system on the same day — one wants to build it locally, one wants to know why it uses eventual consistency instead of a strongly consistent store. Both questions are legitimate; routing the first to the README and the second to an ADR isn't a compromise between them, it's the correct answer to each.

---

Inline comments complete the taxonomy at the smallest scope: tied to specific lines, visible only to a reader already inside the implementation, and unable to describe anything beyond it. Whether a given comment is worth writing at all — and when it rots — is [Ch 30](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md)'s question, not this chapter's; this chapter's job is only to confirm where the inline comment sits relative to the other three: the one artifact scoped below the level of "the whole repository."

### Why Smart Engineers Disagree

The friction isn't about README accuracy — everyone agrees a README should stay current. It's about whether specs and ADRs, once written, should be allowed to change.

One position — evergreening — holds that an out-of-date document is a defective document by definition: if the system's state has moved past the original spec or ADR, the file should be rewritten to match. This optimizes for a single source of truth and a lower cognitive load for a reader who might otherwise mistake an old document for a current one.

The opposing position — structural immutability — treats specs and ADRs as a courtroom record of engineering history: editing a 2024 ADR to fit the system's 2026 shape erases the context that made the original choice correct under 2024's constraints. This position insists that a changed decision gets a new ADR that explicitly supersedes the old one, leaving the original trail intact.

The disagreement tracks organizational scale more than either side credits. A small team with a low total volume of documents can evergreen without much cost — there isn't enough history yet for the rewriting to lose anything that matters. At the scale of a large or long-lived system, structural immutability is the only model that survives: the maintenance burden of retroactively keeping every old spec and ADR aligned with a system that's changed hundreds of times collapses under its own weight, and the two-year-old constraint someone would need to reference is exactly the thing evergreening would have overwritten. Applied consistently, either position can work; the mistake this chapter actually warns against is mixing them — treating one historical artifact as immutable while expecting another with the same purpose to continuously track reality.
