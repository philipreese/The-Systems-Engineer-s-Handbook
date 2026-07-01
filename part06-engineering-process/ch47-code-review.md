# Ch 47 — Code Review

**Prerequisites:** [Local vs. Global Optimization](../part01-systems-thinking/ch08-local-vs-global-optimization.md), [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md), [Issue as Tracking Unit vs. PR as Review Unit](ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md), [Spec-First Development](ch46-spec-first-development.md)

**New vocabulary introduced:** nitpick storm, rubber stamping, review-as-gatekeeping

**Key takeaways:**
- Code review's most durable value is long-term code health and knowledge transfer, not defect detection. Most implementation defects are already caught by types, tests, and static analysis; what a human reviewer uniquely catches is a bad design, unclear code, and missing context — and what the process uniquely builds is shared ownership of the codebase.
- The review standard is improvement, not perfection: approve when a change clearly leaves the codebase better than it found it, even if it isn't how the reviewer would have written it. Perfection-gating stalls PRs and demoralizes authors for no corresponding gain in quality.
- Anything a machine can check — formatting, lint rules, import ordering — must be automated out of review entirely (Principle 8). Human attention is the scarcest resource in the process; spending it on brace placement is a straightforward misallocation.
- Change size and review latency are the two variables that dominate review quality more than any written policy. [Ch 43](ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md) already established that small changes get better review; this chapter adds that turnaround time is a team SLO measured in hours, not days, because delay compounds through rebases and context loss.
- Not every comment carries the same weight. Blocking comments must be labeled as blocking and preference comments must be labeled as optional ("nit:") — conflating the two either stalls good changes over taste or lets real problems slide past as if they were taste.
- Approval is not the same as scrutiny. If approval count becomes a mandatory, tracked metric, approvals will be produced — that's Principle 9 applied to review — but whether real scrutiny happened is a separate question the metric cannot answer.

---

Code review is the most practiced and least examined process in engineering: most teams do it because they always have, without agreeing on what it's actually supposed to accomplish. That question matters more than it looks, because it determines every other policy in this chapter — a team that believes review exists mainly to catch bugs will tolerate slow, adversarial review as the price of thoroughness; a team that understands it as a code-health and knowledge-transfer mechanism will optimize instead for throughput and teaching. This chapter takes a position on that question and on the standard, size, latency, and scope that follow from it. PR sizing and the issue-to-PR mapping were already established in [Ch 43](ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md) and are referenced here, not re-derived. Branching, merge mechanics, commit conventions, and pre-merge CI checks belong to Part VII.

---

### What Code Review Is Actually For

**What it is:** Ranking review's actual purposes honestly, based on what the practice demonstrably delivers rather than its reputation: durable improvement to code health and knowledge transfer, with defect detection real but weaker as a primary justification than most teams assume.

**Why it exists:** Most implementation defects are already caught by type systems, automated tests, and static analysis before a human ever opens a diff — a human reviewer isn't competing with those tools, and isn't especially good at the kind of defect they already catch. What a human uniquely contributes is judgment: recognizing a design that will cause trouble later, code that's technically correct but genuinely unclear, and missing context a machine has no way to evaluate. Review also does something no other part of the process does — it spreads knowledge of the system across more than one person's head and builds a shared sense of ownership over the code, rather than leaving each part of the system understood by only whoever wrote it.

**Options:**
1. **Optimize for code health and knowledge transfer** — treat review as collaborative improvement and cross-training.
2. **Optimize for defect detection** — treat review primarily as manual bug hunting.

**Trade-offs:**

[Strong Recommendation] **Code health and knowledge transfer as the primary framing.** A defect-hunting framing encourages careful examination, but it overestimates what a human reliably catches that automated tooling doesn't, and it tends to produce adversarial reviews — the author and reviewer end up on opposite sides of a search for gotchas instead of collaborating on a shared artifact. A code-health framing can miss the occasional implementation defect that slips past tests too, but it correctly directs review attention toward what only a human can evaluate: whether the design is sound, whether the abstraction will hold up, whether the next engineer will understand it.

**When to choose each:** Every review should still consider correctness — that's table stakes, not optional — but the process itself should be built around long-term code quality and shared understanding, not around the assumption that human eyes are the last line of defense against bugs.

**Common failure modes:** A reviewer spends most of their attention hunting for an isolated off-by-one error while an unclear abstraction or a confusing API shape — the kind of problem that will cost the team real time for years — passes through unremarked, because the review was framed as bug-hunting rather than as an evaluation of the codebase's long-term health.

**Example:** Google's published research on code review at scale identifies code health and knowledge transfer as review's central long-term contributions, with defect detection representing only part of its overall value — a finding that runs against how most teams instinctively justify the process to themselves.

---

### Approve Improvements, Not Perfection

**What it is:** The reviewer's standard for approval: does this change clearly leave the codebase better than it found it — not, is this exactly how the reviewer would have written it.

**Why it exists:** Perfect code doesn't exist, and holding every change to that bar doesn't produce better software — it produces stalled pull requests and authors who learn to dread review. The question that actually matters is comparative: is the system better after this change than before it. A change can be a clear improvement while still leaving room the author or someone else could improve further later.

**Options:**
1. **Improvement standard** — approve any change that clearly improves overall code health.
2. **Perfection standard** — require every raised concern to be resolved before approval.

**Trade-offs:**

[Consensus] **The improvement standard.** This is Google's explicitly published position, and it reflects broad practitioner agreement, not just one company's house style: a perfection standard maximizes the polish of any single change but at a cost that isn't worth it — endless revision rounds, discouraged authors, and delayed delivery of improvements that were already objectively good. The improvement standard accepts that some imperfection will remain in the merged code, in exchange for a review process that keeps moving and keeps contributors willing to submit changes at all.

**When to choose each:** Block a change only when the concern materially affects correctness, maintainability, readability, or the system's ability to evolve. A concern that's really just a stylistic preference should be marked as such — "nit:" is the common convention — and must not block approval on its own.

**Common failure modes:** An author goes through several review rounds satisfying a series of increasingly subjective comments that add up to little measurable improvement, while the conversation quietly shifts from engineering quality to the reviewer's personal taste.

**Example:** Google's engineering practices state the standard directly: a reviewer should favor approving a change if it improves the overall code health of the system, even when it isn't perfect — incremental, continuous improvement is the goal, not an unreachable ideal enforced one PR at a time.

---

### Automate Everything a Machine Can Check Out of Review

**What it is:** Removing formatting, style, and any other mechanically verifiable rule from human review entirely, per Principle 8 — mechanical enforcement is more reliable than human discipline — leaving reviewer attention for correctness, design, readability, and test adequacy.

**Why it exists:** Human attention is the scarcest resource in the review process. Every minute a reviewer spends commenting on brace placement or import order is a minute not spent evaluating whether the underlying design is sound — and unlike a design judgment, a formatting rule is exactly the kind of thing a tool checks perfectly and tirelessly, every time, for free.

**Options:**
1. **Automate mechanical checks** — a formatter and linter enforce style before a human ever sees the diff.
2. **Review everything manually** — reviewers evaluate style and substance together, by hand.

**Trade-offs:**

[Consensus] **Automate mechanical checks; reserve human review for what requires human judgment.** Manual enforcement needs no tooling investment, but it produces exactly the failure mode described below, and wastes reviewer time on repetitive, mechanically resolvable comments. Automated enforcement requires setting up the tooling once, in exchange for a review that starts from 100% of a reviewer's attention available for the things a formatter can't evaluate.

**When to choose each:** If a rule can be checked by a machine — formatting, import order, a straightforward lint rule — it has no place in a human review comment. Reserve human attention for correctness, design, readability, maintainability, and whether the tests actually verify the behavior that matters.

**Common failure modes:** *The nitpick storm.* A reviewer leaves twenty comments on a change — nearly all of them about spacing, trailing commas, or minor naming preferences — and none about the design, the concurrency behavior, or the missing test case that actually mattered. The review looks thorough by comment count, and the reviewer's attention was fully consumed by the time it reached the part of the diff that needed real scrutiny, so the actual risk in the change passes through unexamined.

**Example:** Go's `gofmt`, Python's Black, and JavaScript's Prettier each removed formatting debate from their respective language communities entirely, by making format compliance a mechanical, non-negotiable step before a human ever opens the diff — freeing reviewer attention for the questions a formatter has no opinion on.

---

### Size and Latency Dominate Review Quality

**What it is:** Treating change size and review turnaround time as the two variables that determine review quality far more than any written review policy does.

**Why it exists:** [Ch 43](ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md) already established that review effectiveness degrades sharply as diff size grows — that conclusion applies directly here and isn't re-derived. What this chapter adds is the second variable: latency. A review that sits unanswered for days doesn't just delay one change — the author has already switched contexts and forgotten details by the time comments arrive, the branch has drifted from the base and needs a rebase, and every subsequent review round starts from a colder state than the last, compounding the delay across the whole remaining review cycle.

**Options:**
1. **Fast, frequent review** — turnaround is treated as a team commitment measured in hours.
2. **Slow, occasional review** — reviews are handled whenever a reviewer has spare capacity, with no explicit turnaround commitment.

**Trade-offs:**

[Strong Recommendation] **Fast review turnaround as an explicit team SLO.** Treating review as background work that happens whenever convenient protects an individual reviewer's uninterrupted focus time, but it degrades the whole team's velocity: branches drift, merge conflicts accumulate, and context is lost on both sides of the review. A turnaround SLO measured in hours costs reviewers real, recurring interruptions to their own work, in exchange for keeping the team's overall cycle time — and every author's context — intact.

**When to choose each:** Combine both variables rather than treating them independently: keep changes small per [Ch 43](ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md), and commit to reviewing them within a bounded number of hours, not days. A team that only fixes one of the two still pays for the other.

**Common failure modes:** A small, otherwise well-scoped security patch to a shared library sits unreviewed for several days. By the time a reviewer opens it, three unrelated changes have merged ahead of it, the patch no longer applies cleanly, and the author loses real time to a rebase that has nothing to do with the actual content of the change.

**Example:** Google's own published data shows that fast review turnaround — the majority of changes reviewed well within a business day, smaller ones often within a couple of hours — is a deliberately prioritized organizational commitment, not an incidental benefit of having enough reviewers; it's treated as a velocity multiplier for the whole team, not a courtesy to any one author.

---

### Approval Is Not the Same as Scrutiny

**What it is:** The distinction between a review comment that must block approval and one that's a mere preference, and the related failure mode where approval itself — rather than the scrutiny it's supposed to represent — becomes the thing a team measures and produces.

**Why it exists:** Conflating a blocking concern with a stylistic preference either stalls a good change over taste, or trains authors to treat every comment as equally optional, including the ones that actually matter. Separately, and just as important: an approval is a proxy for scrutiny having occurred, not proof of it. Per Principle 9, the moment a metric like "two approvals required" becomes the mandatory, tracked target, teams will reliably produce two approvals — that says nothing about whether meaningful review happened to earn them.

**Options:**
1. **Severity-aware feedback with a real turnaround SLO** — reviewers label blocking issues as blocking and preferences as optional, and turnaround stays fast enough that reviewers aren't incentivized to skim.
2. **Uniform feedback under approval-count pressure** — every comment is treated the same, and approval counts are tracked as the review-health metric.

**Trade-offs:**

[Strong Recommendation] **Severity-aware feedback, and treat approval counts as a floor, not a quality signal.** Uniform feedback is simpler to apply but leaves authors unable to tell a must-fix from a nice-to-have, producing unnecessary revision cycles or, worse, treating a real problem as optional because it wasn't labeled. Severity-aware feedback requires reviewers to actually exercise judgment about what matters, but it's the only version that lets a change move quickly on its merits instead of on the total count of comments attached to it.

**When to choose each:** Label blocking comments as blocking — they should identify something that materially affects correctness, maintainability, or readability. Label everything else, explicitly, as optional (the common "nit:" convention). Separately, if a team tracks approval count as a gate, treat a monster PR skimmed to a rubber-stamped approval as a process failure, not a completed review — the count was satisfied; the scrutiny wasn't.

**Common failure modes:** *Rubber stamping.* A large, hard-to-review change — often the direct product of the monster-PR failure mode from [Ch 43](ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md) — gets a fast approval from reviewers who confirmed CI was green and skimmed the file list, without actually tracing the logic. The approval-count requirement is satisfied in full; the review that requirement was meant to guarantee never happened. *Review-as-gatekeeping.* A reviewer repeatedly withholds approval not because of a correctness or design concern, but because the implementation doesn't match a personal stylistic preference — asserting authority over the change rather than improving it, and turning a collaborative process into a demonstration of status.

**Example:** An organization requiring two senior approvals on every pull request gets exactly two approvals on a several-thousand-line diff that neither reviewer had time to read carefully — the metric is fully satisfied, and a real concurrency defect ships anyway, because the number the team was tracking was never actually measuring what it was meant to represent.

---

### Why Smart Engineers Disagree on Who Should Review

A separate disagreement, orthogonal to review standard or turnaround, is about who should hold review authority at all: a small set of senior gatekeepers, or the whole team reviewing each other regardless of seniority.

Engineers who favor concentrated review authority argue that codebase cohesion doesn't maintain itself — a small group of people who deeply understand the system's architecture has to be the one consistently checking that new changes don't quietly erode it, especially around the parts of the system where a subtle mistake has wide blast radius. Engineers who favor distributed review argue that concentrating approval power in a few people creates exactly the bottleneck this chapter has spent its latency section arguing against, and that it undermines the knowledge-transfer purpose this chapter opened with — a system understood by only a few senior reviewers is a system where everyone else stays a permanent outsider to it.

Neither position is right everywhere, because the risk of a bad change isn't uniform across a codebase. The workable resolution is to make review authority match blast radius rather than picking one model globally: reserve tighter, senior-gated review for the genuinely hard-to-reverse, wide-blast-radius parts of the system — a shared authentication library, a database migration path, a public API surface — while letting review across ordinary application features stay distributed across the team. That match is the same reversibility-and-blast-radius judgment [Ch 09](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md) applies to decisions generally, applied here to who gets to gate a change instead of just what standard gates it.
