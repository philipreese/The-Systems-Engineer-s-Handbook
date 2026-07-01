# Ch 42 — Issue Tracking: What Makes a Good Issue

**Prerequisites:** [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md), [Complexity Is the Enemy](../part01-systems-thinking/ch02-complexity-is-the-enemy.md), [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md), [Comments: What to Comment, What Not To](../part04-code-organization/ch30-comments-what-to-comment-what-not-to.md)

**New vocabulary introduced:** theme, write-only tracker

**Key takeaways:**
- An issue is a shared engineering artifact, not a personal reminder. It should contain enough context that someone other than its author can complete the work without a clarification thread.
- State the problem before the solution. Recording "cache invalidation serves stale reads after writes" instead of "replace Redis with Postgres" preserves the freedom to discover a better fix during implementation.
- Acceptance criteria convert "done" from an opinion into something another engineer can verify independently. Without them, issues reopen because people disagree about what was actually promised.
- Bugs, features, and chores are different contracts with different required information and different definitions of done. Treating them as one generic type loses information a triager needs.
- One issue should represent one unit of work that can close. An issue with no natural completion point — "improve performance," "clean up the codebase" — is a theme, not an issue, and will sit open indefinitely.
- Creating issues is cheap; maintaining the collection is not. A backlog that is never pruned decays into a *write-only tracker* — closing an issue as "won't do" is a legitimate outcome, not a failure.

---

An issue is the smallest planning artifact in an engineering process: a description of work that someone can understand, perform, verify, and close. Every larger construct in this Part — the pull request in [Ch 43](ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md), the milestone in [Ch 44](ch44-milestone-and-phase-planning.md) — is ultimately an aggregation of issues, and the whole apparatus has to justify its own overhead in [Ch 49](ch49-process-overhead-the-value-threshold.md). If the base unit is poorly defined, every process built on top of it inherits that ambiguity. This chapter defines what makes an issue actionable, how to scope one so it can actually close, and why a healthy tracker requires continual deletion as well as continual creation.

---

### Describe the Problem, Not the Assumed Solution

**What it is:** Writing an issue around the problem being solved and the outcome that would resolve it, rather than immediately prescribing an implementation.

**Why it exists:** The first proposed solution is rarely the best one, and an issue is read long after it is filed — by a different engineer, or by the same one with more context. An issue that records "replace Redis with PostgreSQL" has already discarded every alternative fix before anyone examined the problem it was meant to solve. An issue that records "cache invalidation occasionally serves stale data after a write; stale reads should not occur under normal operation" leaves the door open for whoever picks it up to find a cheaper or more robust fix — including one the original author never considered.

**Options:**
1. **Solution-first** — the issue names the specific implementation to build.
2. **Problem-first** — the issue names the undesired behavior and the outcome that would resolve it, leaving the implementation to whoever does the work.

**Trade-offs:**

[Strong Recommendation] **Problem-first, as the default.** A solution-first issue is immediately actionable and loses nothing when the solution really is externally constrained — migrating off a deprecated library version, complying with a specific regulatory requirement. Outside of that case, it hides the assumptions behind the chosen fix and makes the issue hard to revisit if that fix turns out to be wrong: the team becomes committed to an implementation that no longer makes sense simply because the issue encoded the solution instead of the objective. Problem-first costs a little more to write — sometimes it requires enough design thinking that the issue should reference a spec (see [Ch 46](ch46-spec-first-development.md)) rather than contain the full reasoning itself — but it keeps engineering flexibility intact and makes success easier to evaluate independently of any one implementation.

**When to choose each:** Default to problem-first. Use solution-first only when the implementation truly isn't a design decision — the target library version is fixed by an upstream deprecation, the compliance requirement specifies the exact behavior required.

**Common failure modes:** *The frozen fix.* An issue titled "Replace Redis with PostgreSQL" gets filed, discussed, and eventually implemented — but the underlying stale-read problem was actually caused by a missing cache invalidation call, not the choice of cache backend. Because the issue described a specific migration rather than the symptom, nobody questioned whether the migration was the right fix in the first place.

**Example:** Well-run open-source bug trackers describe observable deficiencies — "search returns stale results after a delete" — rather than prescribing patches. Contributors regularly solve the stated problem in ways the original reporter never anticipated, because the issue left room for that.

---

### Acceptance Criteria Make "Done" Checkable

**What it is:** An explicit statement of the observable conditions that determine whether an issue is complete, distinct from a general description of the desired feature.

**Why it exists:** Without a checkable definition of done, "done" becomes a matter of opinion, and issues reopen because different engineers disagree about whether the promised behavior was actually achieved. Acceptance criteria convert a subjective judgment ("the feature works") into something a second engineer can verify without access to the first engineer's mental model: "a user can reset a password without contacting support," "the API returns HTTP 404 for a nonexistent resource," "the operation completes within two seconds under the stated load."

**Options:**
1. **Implicit completion** — the issue describes the feature; whoever implements it decides when it's done.
2. **Explicit acceptance criteria** — the issue states the specific, observable conditions that must hold for the issue to close.

**Trade-offs:**

[Strong Recommendation] **Explicit acceptance criteria as the default for any shared issue.** Implicit completion is faster to write, but it produces inconsistent interpretations, recurring reopenings, and QA that has nothing concrete to check against. Explicit criteria cost more thought upfront and occasionally need revision as understanding improves mid-implementation, but they make closure objective and testing straightforward — the criteria themselves are close to a test plan.

**When to choose each:** Write explicit criteria for anything a second person might implement, review, or verify — which is nearly all shared work. Criteria should describe observable behavior, not implementation, so that internal refactoring later doesn't change what "done" means.

**Common failure modes:** An issue closes, gets reopened a week later because a different engineer's idea of "works" didn't match the implementer's, closes again with a slightly different behavior, and reopens again. The repeated churn is a symptom that the issue never had a definition of done — only a description of a feature.

**Example:** Teams that define completion around observable system behavior ("returns 404 for unknown IDs") rather than implementation details ("uses a lookup table") can refactor the internals freely without ever having to revisit whether the issue is still closed.

---

### Match Issue Type to the Kind of Work

**What it is:** Treating bugs, features, and tasks/chores as distinct categories with different required information and different definitions of done, rather than one generic issue type.

**Why it exists:** A bug describes existing behavior that is incorrect; a feature introduces new behavior; a task changes engineering assets without changing externally visible functionality. Each needs different information to be actionable and closes under a different condition. A bug report without reproduction steps, expected behavior, observed behavior, and environment turns triage into a guessing game — Joel Spolsky's "Painless Bug Tracking" (2000) made exactly this case over two decades ago, and the structure it recommended is still the standard one. A feature request without acceptance criteria can't be verified as complete. A task without a stated scope never has a moment where it's obviously done.

**Options:**
1. **One generic issue type** for everything, with free-form description.
2. **Distinct issue types** — bug, feature, task/chore — each with its own required fields and definition of done.

**Trade-offs:**

A single generic type is simpler to build tooling around, but it loses information: a triager can't tell from the type alone whether they're looking at a regression, a request, or maintenance work, and has to read the full body every time to figure out which questions even apply. Distinct types require slightly more upfront discipline — someone has to pick the right type and fill in its fields — in exchange for much more consistent issue quality: a bug report that mechanically requires reproduction steps can't be filed without them.

**When to choose each:** Choose the type based on the nature of the work, not organizational convenience or priority-seeking. A bug is incorrect existing behavior, full stop — not "important work" that someone wants bumped up the queue.

**Common failure modes:** Feature requests get filed as bugs to inherit a bug's higher default priority. Maintenance work gets disguised as a feature to get budget allocated to it. Bug reports filed against a generic template omit reproduction steps because nothing forced the reporter to provide them, and the bug sits unreproducible until someone tracks down the reporter for details that should have been in the issue from the start.

**Example:** Spolsky's canonical bug report structure — repro steps, expected behavior, observed behavior, environment — remains the standard shape for a usable bug report because those four fields are exactly what a second engineer needs to reproduce a failure without access to the reporter's machine or memory.

---

### Scope Around One Closeable Unit

**What it is:** Restricting an issue to a single, bounded piece of work that can realistically be completed and closed, as distinct from a *theme* — an ongoing aspiration like "improve performance" or "clean up technical debt" that has no natural completion point.

**Why it exists:** Issues exist to move from open to closed. An issue whose completion criteria are inherently unbounded — "modernize infrastructure" — never actually gets closed; it accumulates unrelated commits under one ticket number indefinitely, and after a few months nobody remembers what "done" would even mean for it. A theme is a legitimate planning concept — an epic, a roadmap category — but it should be decomposed into concrete, closeable issues before it is assigned to anyone, not tracked as a single perpetual ticket.

**Options:**
1. **Atomic bounded issues** — "add request timeout to authentication service," "fix race condition in cache invalidation" — each closeable by a bounded amount of work.
2. **Themes tracked as issues** — "improve performance," "clean up the codebase" — logged directly as a single ticket without decomposition.

**Trade-offs:**

[Strong Recommendation] **Bounded issues for anything assigned to an individual for active work; themes only as an explicit, higher-level category that gets decomposed before assignment.** A bounded issue maps cleanly to a reviewable pull request (the subject of [Ch 43](ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md)), gives predictable, measurable progress, and can be delegated to any available engineer. A theme tracked directly as an issue costs less to file up front, but it produces unclear progress, resists estimation, and rarely finishes — because there was never a defined finish line to reach.

**When to choose each:** If completion can't be objectively recognized, the work needs to be decomposed further. If multiple engineers could work on different parts of it in parallel, it's probably more than one issue already.

**Common failure modes:** *The eternal ticket.* "Clean up the database migration layer" gets assigned to an engineer, who submits ten unrelated pull requests against it over three weeks. Because "cleanup" has no checkable boundary, the ticket stays open indefinitely, distorting milestone metrics and hiding the fact that the actual work is, in practice, already done in spirit if not in ticket status.

**Example:** Large, multi-week architectural transitions — deprecating an API version in a system the size of Kubernetes — are managed through a design document (see [Ch 46](ch46-spec-first-development.md)) plus a set of specific, independently closeable issues ("implement validation webhook for PodSecurityPolicy"), not as one ticket that stays open for the life of the migration.

---

### Enforce Minimum Quality Mechanically, Not by Memory

**What it is:** Using issue templates to require specific fields — reproduction steps and environment for a bug, acceptance criteria for a feature — rather than relying on every reporter to remember what a good issue contains.

**Why it exists:** This is Principle 8 — mechanical enforcement is more reliable than human discipline — applied to process instead of code. If issue quality depends entirely on the reporter remembering the right structure, quality varies by person and by day. A template that requires the fields a triager actually needs makes the well-formed issue the path of least resistance instead of something that depends on everyone's memory.

**Options:**
1. **Manual discipline** — no enforced structure; quality depends on the reporter.
2. **Issue templates** — a per-type template mechanically requires the fields that type needs.

**Trade-offs:**

Manual discipline is flexible and adds no tooling overhead, but produces inconsistent quality and pushes the cost onto whoever triages the issue, who ends up asking the same clarifying questions repeatedly. Templates standardize the information a triager can expect and make automation (routing, labeling) easier, at the cost of becoming bureaucratic if the required-field list grows past what's actually necessary — reporters start spending more effort satisfying the form than describing the problem.

**When to choose each:** Templates earn their cost for recurring categories — bug reports, feature requests, operational incidents — where the required information is predictable. Every mandatory field should have to justify its own existence; a template that demands ten fields to file a one-line typo fix will just get worked around or ignored.

**Common failure modes:** A template starts with three required fields and, over a year, accumulates twelve as different stakeholders each add a field for their own reporting need. Reporters start abandoning the template and pasting free text instead, and the mechanism that was supposed to guarantee quality is now actively discouraging people from filing issues at all.

**Example:** GitHub's `ISSUE_TEMPLATE` mechanism lets a repository require reproduction steps for bug reports while using a different, lighter template for feature requests — mechanical enforcement scoped to exactly the fields that type of issue needs, not a single one-size-fits-all form.

---

### Treat the Backlog as a Living Collection, Not an Archive

**What it is:** Actively closing obsolete, duplicate, and superseded issues rather than letting every filed issue remain open indefinitely.

**Why it exists:** Creating an issue is nearly free. Maintaining the resulting collection is not — every open issue imposes an ongoing cost in prioritization, duplicate detection, and search, and that cost compounds as the collection grows. A backlog that only ever grows becomes a *write-only tracker*: hundreds or thousands of issues that nobody has reviewed in years, that no longer represents what the team actually intends to do, and that developers stop trusting as a planning signal — at which point real work migrates to documents or chat threads and the tracker stops doing its job entirely.

**Options:**
1. **Complete historical backlog** — nothing is ever closed as "won't do"; everything stays open until literally implemented.
2. **Continuously pruned backlog** — obsolete issues, duplicates, and rejected proposals are closed on an ongoing basis.
3. **Automated backlog decay** — tooling auto-archives issues that receive no activity within a defined window.

**Trade-offs:**

[Strong Recommendation] **A continuously pruned backlog, optionally backed by automated decay, over a complete historical archive.** A complete backlog guarantees nothing is forgotten, but stale issues dominate search results, distort prioritization, and eventually the collection stops reflecting the team's actual intentions — the same asymmetry [Ch 39](../part05-testing-strategy/ch39-when-not-to-test.md) made about deleting negative-ROI tests applies here: removal is a legitimate maintenance action, not a failure to keep something around. A pruned backlog requires continuous human attention (or tooling to substitute for it) and occasionally closes something that later turns out to matter, but it keeps the tracker usable as a decision-making tool instead of a graveyard.

**When to choose each:** Treat "won't do" as a healthy, ordinary outcome, not an admission of failure. Reach for automated decay once a backlog is too large for anyone to read and evaluate in a reasonable sitting — roughly the point where triage itself has become a project.

**Common failure modes:** *Backlog bankruptcy.* A team accumulates well over a thousand open issues across several years. The volume makes the backlog impossible to use for milestone planning ([Ch 44](ch44-milestone-and-phase-planning.md)), so the team quietly stops using it altogether and starts tracking active work in documents or chat instead — the tracker still exists, but it no longer does anything.

**Example:** Linear's built-in auto-archive closes an issue that has sat untouched for a set period without a comment, status change, or milestone assignment — a tooling decision that encodes a process philosophy: inactive work should disappear unless someone still actively intends to do it.

---

### Why Smart Engineers Disagree on Tracker Philosophy

The disagreement over issue tracking tools is mostly philosophical, not technical, and it sits on a spectrum with GitHub Issues near one end and Jira near the other. GitHub Issues is a deliberately minimal model — a title, a body, labels, and whatever conventions a team layers on top voluntarily. Jira is a configurable workflow engine — custom fields, required states, permission schemes, transition rules — that lets an organization encode a highly structured process directly into the tool.

Engineers who favor the minimal end argue that most of what a heavyweight tracker enforces is overhead a small, aligned team doesn't need: everyone already shares the missing context, so mandatory fields and workflow gates just slow down filing an issue without adding real information. Engineers who favor the configurable end argue that once an organization scales past the point where everyone shares context — hundreds of engineers, multiple teams, compliance or audit requirements — human discipline alone stops being reliable, and the same mechanical-enforcement argument that justifies issue templates justifies going further: a workflow that refuses to move an issue to "in review" without a linked design doc is Principle 8 applied more aggressively.

Neither position is universally correct, because they're optimizing for different things. A minimal tracker accelerates a small, autonomous team and becomes friction the moment coordination costs actually dominate; a highly configured tracker provides traceability and governance that a small team would experience as pure overhead. The decision is not which tool is better in the abstract, but whether the structure a given tracker imposes removes more coordination cost than it adds for the team using it — the general question of when process earns its keep, which [Ch 49](ch49-process-overhead-the-value-threshold.md) takes up directly.
