# Ch 44 — Milestone and Phase Planning

**Prerequisites:** [Issue Tracking: What Makes a Good Issue](ch42-issue-tracking-what-makes-a-good-issue.md), [Issue as Tracking Unit vs. PR as Review Unit](ch43-issue-as-tracking-unit-vs-pr-as-review-unit.md), [What Engineering Actually Optimizes](../part01-systems-thinking/ch01-what-engineering-optimizes.md), [Complexity Is the Enemy](../part01-systems-thinking/ch02-complexity-is-the-enemy.md), [Decision Frameworks for Trade-offs](../part01-systems-thinking/ch09-decision-frameworks-for-trade-offs.md)

**New vocabulary introduced:** walking skeleton, big-bang integration, 90%-done syndrome

**Key takeaways:**
- A milestone is a coordination and risk-reduction tool, not a deadline ritual. It earns its existence by forcing independently built pieces to integrate and by creating a point where reality gets to correct the plan — not by producing a status report.
- Organize milestone boundaries vertically, not horizontally. The first milestone of any new system should be a *walking skeleton* — the thinnest possible end-to-end slice through every architectural layer — thickened by every milestone after it. A horizontal plan (data layer, then API, then UI) defers integration risk to the point where it is most expensive to discover.
- A milestone can fix its date or its scope, but not both without hiding the cost somewhere else. Fixed time, variable scope is the default that keeps plans honest, because scope is negotiable in small increments and dates are not.
- Order work within a milestone by what's most likely to invalidate the plan, not by what's most pleasant to build. Discovering a fatal assumption on day two is cheap; discovering it in the final week is not.
- Measure milestone progress by working, integrated functionality, not by the percentage of planned tasks marked complete. Task-based progress is a proxy metric, and like any proxy metric made into the target, it stops tracking the thing it was meant to represent — this is the *90%-done syndrome*: a project reports 90% complete for weeks because the unestimated 10% was always the actual risk.
- Big-bang integration — deferring all cross-component integration to a dedicated phase at the end — is a predictable consequence of horizontal planning, not a scheduling accident. It concentrates technical risk at the point in the project when there's the least slack left to absorb it.

---

An issue ([Ch 42](ch42-issue-tracking-what-makes-a-good-issue.md)) is a bounded unit of intent. A milestone groups many issues into a meaningful increment of the system — not to satisfy a calendar or produce a chart for a status meeting, but to create a point where integration actually happens, hidden assumptions get tested, and the plan has to answer to reality. Good milestone planning surfaces uncertainty while it's still cheap to act on. Poor milestone planning doesn't remove that uncertainty — it just postpones it until it's expensive. Specifications for the work being planned are covered in [Ch 46](ch46-spec-first-development.md); release mechanics, tagging, and deployment promotion belong to Part VII.

---

### Milestones Exist to Force Integration, Not to Report Status

**What it is:** Treating a milestone as the point where independently developed pieces are proven to work together, rather than as a checkpoint whose main function is to communicate schedule progress.

**Why it exists:** An issue proves that one isolated piece of work is complete. It does not prove that piece works correctly alongside everything else being built at the same time — a large share of real engineering risk lives precisely in the interactions between components that are each individually correct. A milestone is the mechanism that forces those interactions to actually occur, instead of being assumed away until later.

**Options:**
1. **Integration-focused milestone** — the milestone delivers working, integrated functionality whose behavior can be observed directly.
2. **Reporting-focused milestone** — the milestone exists mainly to summarize which planned tasks have finished.

**Trade-offs:**

[Strong Recommendation] **Integration-focused milestones.** A reporting-focused milestone is simple to communicate up the chain, but it validates nothing technical — tasks can all be "done" while the system they belong to still doesn't function, because nothing forced the pieces to actually meet. An integration-focused milestone surfaces hidden dependencies and validates architecture as a side effect of existing at all, at the cost of requiring continuous integration discipline and occasionally surfacing uncomfortable problems earlier than anyone would like — which is precisely the point.

**When to choose each:** If completing a milestone produces no new, observable system behavior, treat that as a signal that the grouping isn't meaningful yet, regardless of how many tasks it closed.

**Common failure modes:** A team declares a milestone complete because every planned task is checked off, only to discover during the next milestone's work that the pieces don't actually function together — the milestone measured activity, not capability.

**Example:** Distributed systems routinely surface interface mismatches, performance bottlenecks, and deployment assumptions only once independently developed components are actually connected — which is exactly why the connection has to happen inside the milestone, not after it.

---

### Organize Milestones Vertically, Not Horizontally

**What it is:** The choice between a horizontal plan — completing one architectural layer before starting the next (database, then backend, then frontend) — and a vertical plan, where each milestone delivers a thin end-to-end slice of functionality through every layer at once.

**Why it exists:** The planning structure determines when integration risk becomes visible. A horizontal plan defers it to the end, because nothing actually works end-to-end until every layer is finished. A vertical plan exposes integration risk immediately, because the very first milestone has to touch every layer, however thinly.

**Options:**
1. **Horizontal phases** — complete each architectural layer in turn.
2. **Vertical slices** — implement one thin, complete capability through every layer before deepening any of them.

**Trade-offs:**

[Strong Recommendation] **Vertical slices, starting with a walking skeleton.** Horizontal phases align neatly with technical specialization and let each layer's team move fast in isolation, but that speed is an illusion: every architectural assumption connecting the layers stays untested until the final assembly, at which point every hidden dependency surfaces simultaneously. Vertical slices require more upfront cross-layer coordination and force early milestones to accept thin, unoptimized functionality that can feel incomplete to a specialist — but they validate the architecture while there's still runway to change it.

**When to choose each:** For any new system, the first milestone should be a *walking skeleton* — Alistair Cockburn's term for the thinnest possible implementation that exercises every major architectural layer, echoed in Hunt and Thomas's "tracer bullet" in *The Pragmatic Programmer*. Every milestone after it should thicken that skeleton rather than build disconnected subsystems in parallel. Horizontal organization can still exist inside a team's own workflow, but milestone boundaries — the points where the plan claims something is done — should stay vertical.

**Common failure modes:** *Big-bang integration.* A project plans horizontally: the database team, the backend team, and the frontend team each report their phase complete on schedule, then attempt to connect everything for the first time in a dedicated "integration" milestone. The layers immediately collide — concurrency assumptions that held in isolation deadlock under real load, cross-region latency invalidates interface contracts nobody had tested together — and the project enters a retrofit that consumes far more schedule than the integration phase was ever budgeted for. Big-bang integration isn't a scheduling accident; it's the predictable output of a horizontal plan, because the plan never created a point where integration had to happen any earlier.

**Example:** Building a distributed ledger system, a horizontal plan might spend two months modeling the full relational schema before writing any transport code. A vertical plan instead makes Milestone 1 a single end-to-end transaction path — an unindexed write appended to an unreplicated log, exposed through a bare read endpoint — that touches every layer the real system will eventually need. Once that thin path compiles, deploys, and responds, later milestones add indexing, replication, and consensus on top of a foundation already proven to connect.

---

### Fix the Date, Flex the Scope

**What it is:** The recognition that a milestone has two primary constraints — its completion date and its delivered scope — and that under real uncertainty, only one of them can stay fixed.

**Why it exists:** Software development is uncertain enough that unexpected work will appear during any non-trivial milestone. When it does, something has to give: either the scope shrinks to fit the date, or the date slips to fit the scope. Declaring both fixed doesn't remove that trade-off — it just hides it, and the variable that ends up absorbing the overrun by default is quality: skipped tests, rushed review, and debt that nobody explicitly chose to take on.

**Options:**
1. **Fixed scope, variable time** — deliver everything originally planned, however long it takes.
2. **Fixed time, variable scope** — hold the date firm, and cut or defer scope as needed to hit it.

**Trade-offs:**

[Strong Recommendation] **Fixed time, variable scope, as the default.** Fixed scope guarantees feature completeness and suits genuinely non-negotiable obligations — a regulatory deadline, a hardware production window — but it invites the planning fallacy: estimates compress around the best case, deadlines quietly slip, and a schedule that's allowed to slip once tends to keep slipping until it stops being predictive of anything. Fixed time keeps the schedule honest and forces prioritization instead of hope, at the cost of real discipline: someone has to be willing to cut scope rather than quietly extend the date when the milestone runs into trouble.

**When to choose each:** Default to fixed time. Time passes regardless of engineering progress, and scope can almost always be negotiated in smaller increments than a deadline can be moved. Reserve fixed scope for cases where an external constraint genuinely fixes the requirement — and even then, treat the resulting schedule risk as something to name explicitly, not something to plan around optimistically.

**Common failure modes:** A team commits to fixed scope, tracks progress as the raw percentage of sub-tasks completed, and reports 90% done for weeks — because the unestimated final 10% was the actual distributed-systems integration and failure-recovery work, which routinely consumes more effort than the rest of the milestone combined.

**Example:** Basecamp's Shape Up methodology commits to a fixed six-week "appetite" for a cycle and treats scope as the variable: if a team hits a wall that would blow the six weeks, a built-in "circuit breaker" ends the cycle and the unfinished work is dropped or reconsidered from scratch, rather than being allowed to silently absorb another six weeks of schedule.

---

### Order Work by Risk, Not by Comfort

**What it is:** Sequencing the work inside a milestone by which parts are most likely to invalidate the plan, rather than by which parts are easiest or most familiar to build first.

**Why it exists:** The point of a milestone is to find out whether the intended system is actually feasible. The fastest way to answer that is to attack the assumptions most likely to be wrong first — not to build momentum on the parts already known to work.

**Options:**
1. **Risk-first ordering** — implement the work most likely to invalidate the plan before anything else.
2. **Convenience-first ordering** — start with familiar, well-understood, or simply more enjoyable tasks.

**Trade-offs:**

[Strong Recommendation] **Risk-first ordering.** Convenience-first produces fast, visible early progress and easy momentum, but it's an optimization trap: any assumption capable of invalidating the architecture stays hidden at the bottom of the backlog until there's no schedule left to absorb the discovery. Risk-first produces the opposite feeling — slow, unglamorous early progress — in exchange for finding out whether the plan is even viable while there's still time to change it.

**When to choose each:** Identify which assumptions, if wrong, would most change the architecture or the schedule — an unproven third-party integration, a scaling requirement, an unfamiliar consensus protocol — and schedule those first, regardless of how tempting it is to start with the parts that are already well understood.

**Common failure modes:** A team building a real-time tracking system sequences the UI, the user cache, and the notification pipeline first, leaving the actual integration with an external location-data vendor for the final days because it "is just one ticket." The vendor's API turns out to have multi-second tail latency and no batch query support — an architecture-invalidating discovery made after months of polished, now-irrelevant UI work were already sunk into a design the vendor can't support.

**Example:** A team building a high-throughput telemetry pipeline spends its first milestone validating raw disk write throughput and lock contention under simulated peak load — the assumption most likely to invalidate the whole design — before writing any of the authorization layer or reporting dashboards the eventual product also needs, because those pieces are comparatively low-risk and easy to reverse if wrong.

---

### Measure Progress by Working Software, Not Completed Tasks

**What it is:** Evaluating milestone progress by what the integrated system can now observably do, rather than by the percentage of planned tasks marked complete.

**Why it exists:** Task completion is a proxy for progress; integrated, working functionality is the actual objective. This is Principle 9 in its project-management form — a proxy metric, once it becomes the target teams optimize and report against, stops tracking the thing it was meant to represent. A percentage of tasks closed can look identical whether the remaining work is trivial or is the one integration path nobody has attempted yet.

**Options:**
1. **Functionality-based progress** — report what the system can now demonstrably do.
2. **Task-based progress** — report how many planned activities have been marked complete.

**Trade-offs:**

[Strong Recommendation] **Functionality-based progress.** Task-based progress is easy to put in a dashboard, but it's vulnerable to exactly the proxy-metric distortion Principle 9 describes: it rewards checking off easy tasks early and says nothing about whether the hard, integration-heavy work remaining is on track. Functionality-based progress is harder to produce on demand — it requires the system to actually be integrated enough to demonstrate — but it can't be inflated the way a task count can.

**When to choose each:** Wherever a milestone review is possible, demonstrate working functionality rather than summarizing a list of finished tickets. If a status report can't show the system doing something it couldn't do before, treat that as a signal worth investigating, not a formatting inconvenience.

**Common failure modes:** *The 90%-done syndrome.* A project reports 90% complete for several consecutive weeks, because the remaining 10% of the task list was always the unestimated integration, debugging, and dependency work — the part that doesn't compress into a clean percentage — while the completed 90% was largely boilerplate that never represented the project's actual risk.

**Example:** Large software projects have a long, well-documented history of reporting "90% done" for months at a stretch immediately before discovering that a substantial share of the necessary integration work was never represented in the original task breakdown at all.

---

### Why Smart Engineers Disagree on Cadence

A second disagreement surfaces once a team accepts fixed-time milestones: should that fixed time be a short, regular calendar cadence — a two-week sprint, applied uniformly regardless of what's being built — or should milestone boundaries be defined by verified technical events instead?

Engineers who favor a regular calendar cadence argue that human estimation is unreliable enough that the only honest anchor is the clock itself: a fixed two-week boundary forces continuous, small issue slicing, keeps code close to shippable at all times, and gives everyone a predictable rhythm for feedback and re-planning. Engineers who favor event-driven boundaries counter that real integration work doesn't divide neatly into fourteen-day increments — a live database migration or a zero-downtime rewrite of a core module completes when it completes, and forcing it into sprint-sized pieces just means arbitrarily fracturing a cohesive change to satisfy a burndown chart. To this camp, a milestone should be defined by a verified state ("the walking skeleton compiles, replicates state on staging, and passes its stress tests"), not by a date on the calendar.

The productive resolution isn't to pick one exclusively. A regular cadence is a useful operational rhythm — it bounds resource allocation, forces re-planning to happen regularly, and prevents work from drifting indefinitely — but it answers a different question than a milestone does. The milestone itself should still be judged by the integration-focused, vertical-slice criteria this chapter argues for: a cadence tells you when to check in, not whether the thing you're checking on is actually done. Hofstadter's Law is the honest summary of why this matters: a project takes longer than expected, even after accounting for Hofstadter's Law. Milestone planning does not make that uncertainty disappear. It only decides how early the uncertainty gets to show itself, and how much runway remains to act on it once it does.
