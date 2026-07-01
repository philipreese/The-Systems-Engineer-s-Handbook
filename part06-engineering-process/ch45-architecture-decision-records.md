# Ch 45 — Architecture Decision Records (ADRs)

**Prerequisites:** [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md), [Comments: What to Comment, What Not To](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md), [Issue Tracking: What Makes a Good Issue](ch42-issue-tracking-what-makes-a-good-issue.md)

**New vocabulary introduced:** ADR graveyard, retroactive ADR theater, ADR inflation

**Key takeaways:**
- An ADR is not a design proposal and not a discussion transcript. It is the historical record of a decision that has already been made — its purpose is to explain today's decision to the team that inherits it, not to persuade today's team to make it.
- Of the canonical five sections (Title, Status, Context, Decision, Consequences), Context and Consequences carry almost all of the long-term value. Context freezes the constraints that made the decision correct at the time; Consequences records what was knowingly given up, including the alternatives rejected and why — the exact history that, per [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md), prevents a decision from being relitigated by someone who reencounters the same options with none of the history.
- Accepted ADRs are immutable. When a decision changes, a new ADR supersedes the old one and links back to it; the old record is never edited in place. Editing history destroys the exact thing the ADR exists to preserve.
- Not every decision warrants an ADR. Use [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md)'s reversibility-and-blast-radius quadrant directly: hard-to-reverse, wide-blast-radius decisions almost always warrant one; cheap-to-reverse, local decisions almost never do.
- ADRs live in the repository, next to the code they govern, versioned with it — never in a wiki that can drift out of sync with the system it claims to describe.

---

[Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md) established that architectural decisions with high reversal cost or wide blast radius deserve a permanent record, and [Ch 30](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md) ruled that repository-level design rationale belongs in that record rather than in an inline comment. Both chapters deferred the actual format and lifecycle to here — this chapter does not re-derive why ADRs exist; it defines what one looks like, how it moves through its lifecycle, and the judgment for when writing one is worth the effort. Exploring a proposed design before a decision is made belongs to [Ch 46](ch46-spec-first-development.md); the broader taxonomy of README vs. spec vs. ADR as documentation types belongs to Part VIII.

---

### The Canonical Format

**What it is:** The five-section structure Michael Nygard proposed in "Documenting Architecture Decisions" (2011): Title, Status, Context, Decision, Consequences.

**Why it exists:** A consistent structure lets a future reader find the answer to a specific question — what was decided, why, what it cost — without reading a full narrative essay every time. Title and Status are bookkeeping: what this record is about and where it stands in its lifecycle. Decision states the choice plainly. Context and Consequences, covered in the next two sections, are where nearly all of the long-term value lives.

**Options:**
1. **The canonical five-section format** — lightweight, fixed, minimal.
2. **An expanded, organization-specific template** — additional sections for stakeholders, cost estimates, compliance sign-off, and so on.

**Trade-offs:**

[Strong Recommendation] **Start with the canonical format and resist expanding it.** The canonical format is concise enough to actually get written consistently, and it's widely recognized enough that engineers moving between projects already know how to read it. An expanded template can capture more organizational context and integrate with formal governance processes, but every additional required section raises the cost of writing an ADR — and, per the mechanical-enforcement principle, a template that demands more than the decision actually needs will get worked around rather than filled in honestly.

**When to choose each:** Add a section only once the canonical five have repeatedly failed to answer a question people actually ask. Don't add a section speculatively because it might be useful someday.

**Common failure modes:** A template starts at five sections and, over a few years, accumulates a dozen as different stakeholders each add a mandatory field for their own reporting need. Writing an ADR starts to resemble filling out a compliance form, and engineers either avoid writing them or fill the new fields with boilerplate that satisfies the form without adding real information.

**Example:** ThoughtWorks' Technology Radar helped popularize the lightweight five-section version specifically because it stayed concise enough that teams would actually write one instead of treating it as a chore to be deferred. Tooling built around the same shape — `adr-tools`, MADR — grew directly from Nygard's original format rather than reinventing it.

---

### Context Captures What Was True at Decision Time

**What it is:** The section of the ADR that records the constraints, assumptions, and facts that existed at the moment the decision was made — not the decision itself, and not how those constraints look today.

**Why it exists:** Architectural decisions are rarely universally correct; they're correct given a specific set of constraints that were true at the time. A future engineer can only judge whether a decision still applies once they know what those original constraints actually were. An ADR that records "we chose SQLite" without recording that the deciding constraint was single-user, embedded, offline-capable deployment gives a future reader nothing to evaluate — the decision looks arbitrary the moment the constraint that justified it is no longer visible.

**Options:**
1. **Rich context** — explain the specific forces (scale, team size, deadline, regulatory requirement, existing infrastructure) that drove the decision.
2. **Minimal context** — record only the conclusion, with little or no explanation of the constraints behind it.

**Trade-offs:**

[Strong Recommendation] **Rich context.** Minimal context is faster to write, but it strips the ADR of the one thing that makes it re-evaluable later: without the original constraints, a future team can't tell whether the decision still holds or was only ever correct under conditions that no longer exist. Rich context costs more thought at write time — someone has to actually articulate the forces at play instead of just announcing a conclusion — but it's what turns the record from an announcement into something a future engineer can reason from.

**When to choose each:** Include every constraint that materially influenced the decision — a real scaling number, a regulatory requirement, an existing dependency the team was locked into. Don't try to document the discussion that led there; a meeting transcript is not context.

**Common failure modes:** *Retroactive ADR theater.* A production incident traces back to a poorly chosen sharding strategy. To satisfy a post-mortem review, the team writes an ADR after the fact — but the context it records is a cleaned-up, idealized version of the reasoning, quietly omitting the flawed assumption that actually drove the original choice. The record now documents the outcome, not the real reasoning, and the next team to hit the same flawed assumption gets no warning from it.

**Example:** An ADR choosing SQLite over PostgreSQL only means something if its Context section states that embedded, single-process deployment with no operational database team was the deciding constraint — without that, "we chose SQLite" reads as an arbitrary preference rather than a reasoned trade-off a future team can judge on its own merits.

---

### Consequences Record the Cost, Including What Was Rejected

**What it is:** The section that records what the team knowingly accepted by making the decision — benefits, costs, limitations, and specifically the alternatives that were considered and rejected, along with why.

**Why it exists:** [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md) already made the case: recording rejected alternatives is what stops a future engineer from re-litigating an option that was already tried and ruled out, without any memory that it was. The decision alone is only half the record — the rejected alternatives are why that decision is the one that survived.

**Options:**
1. **Full consequences** — document benefits, accepted costs, and specifically which alternatives were rejected and why.
2. **Decision only** — treat the ADR as an announcement of the outcome, with no record of what else was considered.

**Trade-offs:**

[Strong Recommendation] **Full consequences, every time.** A decision-only record is more concise, but it guarantees the same options get re-evaluated from scratch every time someone new encounters the same problem — the ADR exists but doesn't do the one job that justifies its cost. Recording the accepted trade-offs and rejected alternatives costs a few more paragraphs, in exchange for genuinely discouraging repeat debates and making the compromise the team accepted visible instead of implicit.

**When to choose each:** Always write full consequences for anything that clears the bar for having an ADR at all (see below) — a decision significant enough to record is significant enough to explain what it cost.

**Common failure modes:** Engineers propose "a new idea" years later that was already evaluated and rejected under the same constraints, because the original ADR recorded only the winning choice and never mentioned the alternative had been considered at all.

**Example:** A team choosing a monolith over a service decomposition should record not just the simplicity gained, but the scaling ceiling and organizational coordination limits it knowingly accepted — so that when those limits are eventually hit, the response is "we knew this was coming" rather than a scramble to rediscover why the monolith was chosen in the first place.

---

### Accepted ADRs Are Immutable — Supersede, Don't Edit

**What it is:** The lifecycle an ADR moves through — Proposed, Accepted, and eventually Deprecated or Superseded — with the rule that once an ADR reaches Accepted, its content never changes again.

**Why it exists:** An ADR is a historical record, not a living summary of the current architecture. When architectural thinking changes, the historical fact isn't that the old decision quietly became something else — the fact is that a new decision replaced it at a specific point in time. Editing an accepted ADR in place erases that distinction and rewrites what the team actually believed when the original decision was made.

**Options:**
1. **Immutable, superseding** — accepted ADRs never change; a new ADR is written, marked as superseding the old one, and the old one is marked Superseded with a link forward.
2. **Editable in place** — the existing ADR is updated directly so the directory always reflects current architecture.

**Trade-offs:**

[Strong Recommendation] **Immutable, superseding records.** Editing in place keeps the documentation directory looking clean and current, but it destroys exactly the information an ADR exists to preserve: a reader has no way to know the system used to operate under different assumptions, and legacy code built against the old decision starts looking like unexplained non-compliance rather than a valid response to constraints that were true at the time. Immutable records mean a reader occasionally has to follow a chain of two or three linked documents to find the current state, but every one of those documents remains an accurate snapshot of what was believed and why, at the moment it was written.

**When to choose each:** Treat every accepted ADR as permanently locked. When circumstances change, write a new, numbered ADR that references the one it replaces, and update the old record's Status to point forward — never rewrite its Context or Decision sections.

**Common failure modes:** A team edits an existing ADR in place to reflect a shift from one messaging technology to another, without creating a new record. A third of the production services still run on the original technology; a new engineer reads the edited ADR, assumes those services are out of compliance with team standards, and starts an unnecessary, unbudgeted migration — because the record no longer shows that the older services were built correctly under constraints that genuinely applied to them at the time.

**Example:** A repository moving its event streaming from a self-hosted system to a managed one adds a new, sequentially numbered ADR whose Status reads "Accepted; supersedes ADR-0012," while the original ADR-0012 is updated only in its Status field to read "Superseded by ADR-0034." The original Context explaining why the self-hosted choice was correct at the time it was made stays exactly as written.

---

### When an ADR Is Warranted

**What it is:** The judgment call of which decisions actually earn a permanent record, applying [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md)'s reversibility-and-blast-radius framework directly rather than treating every technical choice as equally deserving of one.

**Why it exists:** Writing an ADR costs engineering effort, and that cost is only worth paying for decisions a future engineer is genuinely likely to need to revisit or re-evaluate. Applying the reversibility-and-blast-radius quadrant that [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md) already established gives a concrete answer instead of a vague sense that "important" decisions deserve documentation.

**Options:**
1. **Record only high-impact architectural decisions** — persistence technology, distributed communication model, deployment architecture, authentication strategy, service decomposition boundaries, and similarly hard-to-reverse, wide-blast-radius choices.
2. **Record every technical decision** — treat the ADR directory as a comprehensive engineering journal.

**Trade-offs:**

[Strong Recommendation] **Selective recording, gated by reversibility and blast radius.** Recording everything produces a comprehensive history in principle, but in practice it drowns the decisions that actually matter in volume — a reader looking for why the system uses a particular persistence model has to wade through ADRs about internal folder layout and library choice to find it. Selective recording requires real judgment about where the threshold sits, but it keeps the collection small enough that people actually consult it, which is the only thing that makes an ADR directory worth maintaining at all.

**When to choose each:** Decisions that are both expensive to reverse and broadly consequential — the ones in the top-right of [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md)'s quadrant — almost always warrant an ADR. Local implementation choices, internal naming, and routine library usage almost never do, regardless of how much debate they generated in the moment.

**Common failure modes:** *ADR inflation.* A team lowers its threshold until routine choices — which validation library to use, how to lay out an internal folder — receive full ADRs alongside genuinely architectural ones. The directory grows past the point anyone can skim, the handful of decisions that actually matter get lost in the noise, and engineers stop checking the directory at all, at which point the practice has stopped doing its job regardless of how many records exist.

**Example:** Choosing a database's persistence model, a distributed system's consistency guarantees, or an authentication strategy — the kind of decision that's expensive to reverse and affects the whole system — warrants an ADR nearly every time. Choosing how to name a helper function inside one module does not, no matter how long the debate about it ran.

---

### Store ADRs With the Code They Govern

**What it is:** Keeping ADRs inside the repository they describe — conventionally in a `docs/adr/` or `docs/decisions/` directory of numbered files — versioned alongside the code, rather than in a separate wiki or documentation portal.

**Why it exists:** Architecture evolves together with the code implementing it. Storing decision records anywhere other than the repository — a wiki, a shared drive, a project management tool — separates the record's lifecycle from the code's lifecycle, and the two drift apart the moment nobody is deliberately keeping them synchronized.

**Options:**
1. **In-repository storage** — a numbered directory of markdown files, versioned with the code.
2. **External wiki or documentation portal** — a centralized system maintained separately from version control.

**Trade-offs:**

[Strong Recommendation] **In-repository storage.** A wiki offers easier access for non-technical stakeholders and centralized cross-team visibility, but it sits outside the pull request workflow entirely — nothing forces an engineer to open it during review, so it decays quietly while the code moves on without it. In-repository storage means an ADR change can be reviewed in the same PR as the code change it justifies, and the record travels automatically with every clone, fork, and branch — at some cost to non-technical stakeholders who now need a different way to see the same information.

**When to choose each:** Store ADRs in the repository for any codebase that will be maintained past its initial build. Reserve external documentation systems for genuinely non-technical audiences — organizational charts, product roadmaps — that were never going to consult architecture records in the first place.

**Common failure modes:** *The ADR graveyard.* An organization sets up a dedicated space in a wiki tool for architecture records and dutifully logs several dozen decisions there to satisfy a process checkpoint. Because the wiki sits outside the normal review workflow, no engineer ever links to or consults an entry during an actual pull request, the codebase drifts steadily away from what the records describe, and within a couple of quarters the collection is a graveyard nobody trusts enough to open.

**Example:** A `docs/adr/` directory at the root of a repository lets a new engineer clone the codebase and read every architectural decision that shaped it directly from the terminal, in the same version-controlled history as the code itself — no separate login, no separate tool, and no risk that the two have quietly diverged.

---

### Why Smart Engineers Disagree

Most of the disagreement about ADRs isn't about whether recording major architectural decisions is worthwhile — it's about where the threshold for "major" actually sits.

One camp argues for a strict threshold, gated tightly to [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md)'s hardest-to-reverse, widest-blast-radius quadrant only: changing a persistence paradigm, adopting a new runtime, altering a tenant-isolation model. They point to ADR inflation as the real risk — the moment routine, cheap-to-reverse choices start getting the same treatment, the directory's signal gets diluted and people stop reading it at all. The other camp argues for a much lower bar: human memory is unreliable, and what looks like a cheap, local choice on day one can quietly compound into a load-bearing architectural assumption a few years later. To this camp, a three-minute lightweight record (the appeal of formats like MADR specifically) costs so little that erring toward recording more is worth the marginal noise.

Both sides are reasoning from the same failure mode — lost context — and disagreeing only about which direction is riskier: too little signal, or too much noise drowning the signal that exists. The practical resolution isn't to pick a universal threshold and apply it uniformly forever. It's to keep the friction of writing one low (lightweight templates, tooling that automates the numbering and cross-linking) while holding the line that the *directory* stays a high-signal collection — which means the threshold question is really a proxy for whether ADRs are still being read. If ADRs are being cited in reviews and onboarding, the threshold is calibrated correctly regardless of which camp set it. If they aren't, the collection has already failed, whether from too few records or too many.
