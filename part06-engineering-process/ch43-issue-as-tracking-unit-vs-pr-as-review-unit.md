# Ch 43 — Issue as Tracking Unit vs. PR as Review Unit

**Prerequisites:** [Issue Tracking: What Makes a Good Issue](ch42-issue-tracking-what-makes-a-good-issue.md), [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md), [Complexity Is the Enemy](../part01-systems-thinking/ch02-complexity-is-the-enemy.md), [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md)

**New vocabulary introduced:** unit of intent, unit of review, monster PR

**Key takeaways:**
- An issue and a pull request answer different questions and have different natural sizes. The issue is the *unit of intent* — why this work exists and what outcome closes it. The PR is the *unit of review* — a bounded diff a human can actually evaluate for correctness.
- The natural mapping between them is one-to-many, not one-to-one. An issue closes when its acceptance criteria are met, regardless of how many PRs it took to get there.
- Review effectiveness degrades sharply as diff size grows — this is an empirical finding, not a matter of reviewer discipline. The burden of keeping diffs small belongs on the author, not the reviewer.
- Forcing a 1:1 mapping produces one of two failure modes: the *monster PR* that reviews an entire issue in one unreviewable diff, or a fragmented issue tracker sliced to match convenient diff boundaries, which destroys the tracker's value as a record of intent.
- Incremental PRs, feature flags, and stacked PRs are the standard techniques for splitting implementation into reviewable steps while keeping one stable issue as the record of intent.
- Every PR should reference the issue that motivated it. An issue with no linked activity, or a PR with no linked issue, is a signal worth noticing — the alternative is a PR-only workflow where the diff explains how but nothing records why.

---

[Ch 42](ch42-issue-tracking-what-makes-a-good-issue.md) established what makes a single issue actionable. This chapter draws a distinction most teams never make explicit, and pay for that omission later: the issue and the pull request are different units, sized for different purposes, and the mapping between them is naturally one-to-many. Confusing them produces planning artifacts that no longer represent real work, or code reviews too large for anyone to review effectively. The mechanics of branches and merging are covered in Part VII; what reviewers should actually look for during review is covered in [Ch 47](ch47-code-review.md). This chapter is only about what unit each artifact should be sized to.

---

### Size the Mapping Around One Issue, Many PRs

**What it is:** The choice between forcing every issue to close via exactly one pull request, and letting a single issue be satisfied by however many PRs its implementation naturally requires.

**Why it exists:** An issue is a unit of intent: it defines a problem and the acceptance criteria that resolve it, and it stays open until that outcome is achieved — however many implementation steps that takes. A PR is a unit of review: it exists to be verified by a human within one sitting, and its natural size is bounded by what a reviewer can actually hold in their head at once. These two objectives rarely produce the same boundary. Forcing them to coincide means one of them loses: either the issue's scope gets stretched to whatever a reviewer can handle, or the PR's size gets stretched to whatever closes the issue.

**Options:**
1. **1:1 mapping** — every issue is closed by exactly one PR.
2. **1:N mapping** — an issue is closed once its acceptance criteria are satisfied, across as many PRs as the implementation needs.

**Trade-offs:**

[Strong Recommendation] **1:N as the default for any non-trivial work.** A 1:1 mapping keeps bookkeeping simple — the issue auto-closes when its one PR merges — but it forces exactly the choice described above. If the issue represents real work, the corresponding PR grows until it's unreviewable. If a team avoids that by keeping PRs small, it ends up fracturing the issue tracker into artificial slices that match diff boundaries instead of real intent, and the tracker stops representing anything a planner or stakeholder can use. A 1:N mapping preserves the integrity of both units — the issue stays a stable statement of intent, the PR stays sized for review — at the cost of extra bookkeeping: cross-referencing PRs against the issue and tracking which parts are done.

**When to choose each:** 1:1 is reasonable for trivial, single-layer changes — a dependency bump, a one-line hotfix, a documentation typo — where the entire change already fits comfortably in one review. For anything that spans more than one architectural layer, or that takes more than a sitting to implement, default to 1:N and keep the issue as the single point of reference.

**Common failure modes:** *The monster PR.* A team enforcing strict 1:1 mapping implements a multi-region payment routing change — database models, network client integration, retry queues, and configuration — as a single diff of several thousand lines. It sits unreviewed for weeks because no reviewer can safely hold the whole thing in context at once, and by the time anyone attempts a serious review, the base branch has already drifted underneath it. *The fragmented backlog.* To avoid monster PRs under a 1:1 rule, a team instead splits one cohesive feature into dozens of issues — "create user table," "add name column," "expose name field via API" — each one matching a convenient diff instead of a real unit of intent. The tracker becomes a mirror of implementation steps and stops being usable as a roadmap.

**Example:** A migration of an application's caching layer from a local in-process store to a distributed Redis cluster is tracked as one issue with clear acceptance criteria (cache reads and writes go through Redis; local state is removed; no correctness regression under load) and implemented across a sequence of PRs: introducing the Redis client adapter, dual-writing to both stores behind a toggle, cutting reads over, then deleting the legacy code. One issue, four PRs, each independently reviewable.

---

### Review Effectiveness Degrades as Diffs Grow

**What it is:** The empirical observation that reviewers detect fewer defects per line, and give less scrutiny overall, as the size of a single review grows — not because reviewers become careless, but because human attention does not scale linearly with diff size.

**Why it exists:** A review unit has to be sized for human comprehension, not for project convenience. Past a certain size, a reviewer physically cannot hold the whole change in working memory at once, and the review shifts from actual verification toward something closer to trust: skimming, checking that CI is green, and approving.

**Options:**
1. **Small reviews, frequent turnaround** — changes are split so each one stays within what a reviewer can evaluate carefully in one sitting.
2. **Large reviews, infrequent turnaround** — changes accumulate into a bigger diff before being submitted for review.

**Trade-offs:**

[Consensus] **Small, frequent reviews over large, infrequent ones.** This is not a matter of taste — it's one of the more consistently reproduced findings in empirical software engineering. The SmartBear/Cisco code review study found that defect detection falls off sharply once a single review passes roughly 200–400 lines, and Google's own published research on code review at scale ("Modern Code Review: A Case Study at Google," Sadowski et al., 2018) documents that keeping change sizes far smaller than typical industry norms, paired with fast review turnaround, is a deliberate, institutionally reinforced practice rather than an accident of tooling. Large reviews reduce the number of times an author has to ask for review, but that saving is illusory: review quality collapses well before the diff gets large enough to save meaningful coordination effort.

**When to choose each:** The responsibility for splitting a change belongs to the author before submission, not to the reviewer after the fact. If a reviewer cannot reasonably read and understand the entire diff in one sitting, the change is too large regardless of how logically connected its parts are.

**Common failure modes:** A large diff gets a handful of superficial line comments and an approval within minutes — the reviewer has effectively verified that the pipeline is green, not that the logic is correct. A hidden concurrency bug or a missing index allocation passes unnoticed because no one had the attention budget left to trace through the less obvious paths once the diff crossed a few hundred lines.

**Example:** An 800-line PR alters a core financial allocation loop. Because of its size, the reviewer skims it, confirms CI passes, and approves within minutes. A race condition in the new concurrency handling ships to production and causes silent data corruption — a defect that a reviewer working through a 100-line version of the same change, one step at a time, would have had a realistic chance of catching.

---

### Split Implementation Without Splitting Intent

**What it is:** The concrete techniques that let a team divide an issue's implementation into small, reviewable PRs while the issue itself stays a single, stable statement of intent: incremental PRs, feature flags, and stacked PRs.

**Why it exists:** Knowing that small PRs are better than large ones is not the same as knowing how to produce them without either breaking the main branch or exposing half-finished functionality to users. Each of these three techniques solves that problem in a different way, and each fits a different shape of dependency between the steps.

**Options:**
1. **Incremental PRs** — each PR leaves the system fully working; later PRs build on top.
2. **Feature flags** — implementation is merged behind a toggle before the functionality is exposed, so incomplete but inert code reaches the main branch safely.
3. **Stacked PRs** — a sequence of dependent branches, each reviewed against the one before it, for changes where a later step genuinely cannot be understood or validated without the exact code of the step before it.

**Trade-offs:**

Incremental PRs are the simplest option and require no extra machinery, but only work when the underlying work can actually be decomposed into steps that each leave the system functional — not every change decomposes that cleanly. Feature flags handle the case where the work can't be exposed incrementally (a UI redesign, a new API surface) by merging real code before it's turned on, at the cost of a toggle that has to be removed later — a flag left in the codebase after its purpose is served is dead configuration that erodes the ability to reason about the system's actual behavior. Stacked PRs handle tightly sequential work where each step's diff only makes sense in the context of the one before it, giving reviewers small individual diffs without hiding anything behind a flag, at the cost of real git discipline: an upstream fix has to be rebased down through every dependent branch, and without tooling support that quickly turns into a maintenance burden of its own.

**When to choose each:** Use incremental PRs whenever the work can be sliced into steps that each leave the system working. Use feature flags when the functionality genuinely can't be exposed piece by piece. Use stacked PRs when later steps are written against the exact shape of earlier ones and can't be reviewed in isolation.

**Common failure modes:** A team adopts feature flags without a pruning discipline (see [Ch 49](ch49-process-overhead-the-value-threshold.md)), and over several quarters accumulates dozens of stale, unmaintained flag conditions. Nobody can reason confidently about which code paths are actually live, and an old, forgotten flag evaluating unexpectedly causes a production incident. Separately, a team avoids merging anything until an entire stack of dependent PRs is complete, defeating the purpose of stacking — the branches grow long-lived and diverge from the base exactly as much as a single monster PR would have.

**Example:** The popularity of stacked-PR tooling such as Graphite reflects how common tightly sequential work actually is: a chain of dependent branches, each reviewed on its own, with tooling handling the rebase propagation that would otherwise make maintaining the stack impractical by hand.

---

### Link Every PR to the Issue It Serves

**What it is:** The discipline of having every pull request explicitly reference the issue that motivated it, so the code's *how* and the issue's *why* remain connected after the PR merges.

**Why it exists:** The PR explains how a change was implemented. It doesn't, on its own, explain why the change was needed, what alternatives were considered, or what constraint made the chosen approach the right one — that context lives in the issue. Without an explicit link between them, that connection has to be reconstructed by hand later, by searching commit messages, review comments, and timestamps for a thread that may no longer be traceable at all.

**Options:**
1. **Explicit linkage** — every PR references its motivating issue, mechanically where possible (GitHub's closing keywords: `fixes #123`, `closes #123`).
2. **Implicit linkage** — the relationship between a PR and its motivating work is left for a future reader to infer.

**Trade-offs:**

[Strong Recommendation] **Explicit linkage by default.** The cost is negligible — one line in a PR description — against a real, recurring cost on the other side: an engineer investigating a production incident who can find exactly how a value was changed but not why, because the PR description just said "optimize socket pooling" and the issue that explained the actual constraint was never referenced. Explicit linkage also gets automatic status updates for free: a merged PR referencing `fixes #123` closes the issue without anyone updating two systems by hand.

**When to choose each:** Use explicit linkage for any change against a codebase that will be maintained past the current sprint. Implicit linkage is only tolerable for disposable, short-lived work — a prototype spike, throwaway tooling — that no one will need to trace back to intent later.

**Common failure modes:** *The PR-only workflow.* A team gradually stops filing issues at all and relies on PR descriptions as the sole record of both intent and implementation. Six months later, an engineer can see that a timeout was changed from five seconds to two hundred and fifty milliseconds, but the reasoning behind that specific number was never written down anywhere durable, and reconstructing it becomes a guessing exercise rather than an archaeology one.

**Example:** GitHub's issue-closing keywords let a merged PR automatically close the issue it references, giving teams bi-directional traceability — from issue to implementation and back — without anyone maintaining that link by hand. Where a decision itself needs to survive independently of any single issue or PR — a rejected alternative architecture, a rationale that will matter again in two years — that belongs in an ADR, not a PR description (see [Ch 45](ch45-architecture-decision-records.md)).

---

### Why Smart Engineers Disagree on Where Intent Should Live

Once a team accepts the one-issue-many-PRs model, a second disagreement surfaces: should the durable record of *why* a change was made live in the issue tracker, or in version control itself — commit messages and PR descriptions?

Engineers who favor the issue tracker point to cross-functional visibility: product, security, and support all need to see and search engineering context without reading diffs, and an issue tracker is built for that in a way a commit log is not. Engineers who favor version control point to durability: companies migrate issue-tracking vendors, reorganize, or lose access to old boards, and when that happens the tracker's history goes with it, while the git log is comparatively permanent and travels with the code itself.

Both concerns are legitimate, and the practical answer isn't to pick one side. Use the issue tracker for what it's actually good at — coordinating active work, prioritization, and cross-team visibility during a project's live lifecycle. But for a decision whose reasoning needs to survive independently of any tracker — why an alternative architecture was rejected, why a specific constraint was chosen — write it down where it can't be lost to a vendor migration: in the commit history, or as a standalone Architecture Decision Record ([Ch 45](ch45-architecture-decision-records.md)). The issue tracker is the right home for short-lived coordination; permanent rationale belongs somewhere that outlives the tool.
