# Ch 53 — Branch Naming and Lifecycle

**Prerequisites:** [Issue Tracking: What Makes a Good Issue](../part06-engineering-process/ch42-issue-tracking-what-makes-a-good-issue.md) (write-only tracker), [Process Overhead: The Value Threshold](../part06-engineering-process/ch49-process-overhead-the-value-threshold.md), [Branching Strategies](ch50-branching-strategies.md), [Squash vs. Preserve History](ch52-squash-vs-preserve-history.md)

**New vocabulary introduced:** None — the failure mode this chapter's lifecycle argument rests on is the write-only tracker, already defined in Ch 42.

**Key takeaways:**
- Branch naming is a weak convention layer; branch lifecycle is a real system-design constraint. Don't spend more thought on naming than the near-bikeshed it actually is — the lifecycle discipline is where this chapter's real argument lives.
- [Strong Recommendation] A hybrid naming scheme — type prefix plus issue key (`fix/ENG-4321-auth-timeout`) — earns its minor friction by letting CI and platform webhooks parse a branch name and automatically transition the linked ticket's state. A branch name is not a substitute for the issue it should reference, and shouldn't try to carry state that belongs in the tracker (Ch 42).
- [Strong Recommendation] Automatic deletion of a branch on merge, mechanically enforced through the hosting platform's setting, is the correct default — Principle 8 applied directly: don't leave to manual discipline what a platform toggle can enforce for free.
- An unpruned branch list degrades exactly like an unpruned issue backlog: the write-only tracker failure mode (Ch 42), expressed in Git's namespace instead of the issue tracker's. Cheap to add to, expensive to prune, and eventually untrusted by everyone who has to search it.
- Deleting a merged branch's ref doesn't delete history — the commits already live on the target branch. Retaining branches "for safety" is solving a problem tags and releases (Ch 56) already solve, with a mechanism actually designed for long-term retention.

---

This chapter covers two mechanically independent concerns: how a branch is named, and what happens to it after it merges. It does not re-litigate branching topology or why branches should be short-lived (Ch 50 — this chapter assumes that conclusion), commit message format (Ch 51), or history-preservation policy at the merge gate (Ch 52). It also doesn't redefine what makes an issue well-scoped (Ch 42) — a branch name can reference an issue, but the issue's quality is that chapter's argument, not this one's.

### Branch Naming: Convention, Not Architecture

**What it is:** A prefix-based scheme for labeling branches by change type and, optionally, by the issue that motivates them — for example, `fix/ENG-4321-auth-timeout` or `feature/INFRA-892-redis-cluster-migration`.

**Why it exists:** To make a branch list scannable and weakly machine-parseable without requiring an external system to look anything up. A well-formed name lets a CI webhook or platform integration read the embedded issue key and automatically move that ticket's state — a real but modest mechanical payoff, not a substitute for the tracking system itself.

**Options:**
1. **Descriptive prose** — a free-text summary of the branch's goal (`fix-broken-auth-timeout-on-login-page`).
2. **Type prefix only** (`feature/`, `fix/`, `chore/`) — categorizes the change without linking to any tracker.
3. **Type prefix + issue key** — categorizes the change and embeds a token an external system can key off of.

**Trade-offs:** Prose naming is unconstrained and reads well locally, but eliminates any programmatic link to an issue tracker and invites both bikeshedding over the name and lazy, contentless names (`test-code`) at the other extreme. Type-prefix-only naming improves scannability but adds no structural meaning beyond convention. Type prefix plus issue key adds real mechanical value — CI can parse it — at the cost of a small amount of friction every time a branch is created.

**When to choose each:** Prose naming is fine for personal, unpushed, throwaway work. Type-prefix-only naming is adequate for small teams without a formal issue tracker. Type prefix plus issue key is the right default for any team already running Ch 42's issue-tracking discipline, since the marginal cost of embedding the key is near zero and the automation payoff (auto-transitioning ticket state, filtering unmerged work by type) is real.

**Common failure modes:** The unlinked phantom — a branch named `fix-stuff`, with no issue reference, sits unmerged for days while its author is unavailable; when a related bug surfaces, nobody has a path back to why the change was made or what it was meant to fix, because the name carries no traceable context. The inverse failure is asking naming to do too much: a branch name that grows into a mini-description of the change, or that's relied on to encode status information (blocked, needs-review, WIP) that belongs in the issue tracker, not in a string a human has to keep editing.

**Example:** A branch pushed as `feature/INFRA-892-redis-cluster-migration` lets the hosting platform read the `INFRA-892` token and automatically move that ticket from "In Progress" to "In Review" the moment the corresponding PR opens — a small piece of automation that only works because the naming convention was followed consistently.

### Branch Lifecycle: Ephemeral by Default, Deleted on Merge

**What it is:** Treating a branch as a disposable execution artifact that exists only until its change integrates, and deleting it immediately afterward rather than retaining it "just in case."

**Why it exists:** Branch creation is free; branch cleanup is not, unless it's automated. Left to manual discipline, cleanup degrades exactly the way an unpruned issue backlog degrades (Ch 42's write-only tracker): branches accumulate because nobody is incentivized to delete one, until the branch list stops reflecting what's actually being worked on and stops being trusted as a navigation tool.

**Options:**
1. **Manual deletion** — an engineer or release manager deletes branches by hand, on their own schedule.
2. **Automatic deletion on merge** — the hosting platform deletes the remote branch ref the moment its PR merges.
3. **Retention** — branches are kept indefinitely, treated as a backup or audit trail.

**Trade-offs:** Automatic deletion keeps the branch namespace aligned with active work at zero ongoing cost, and reinforces the short-lived-branch culture Ch 50 already argues for — its only real cost is requiring trust that the target branch's history (Ch 52) actually preserved what mattered, since recovering a deleted branch afterward means digging through `reflog` rather than checking out a live ref. Manual deletion relies on discipline that reliably fails at scale — nobody's job is to remember it, so it doesn't happen. Retention avoids that failure mode entirely but guarantees the opposite one: a branch list that grows without bound and stops being useful to search.

[Strong Recommendation] Automatic deletion on merge, enforced at the platform level (GitHub's and GitLab's "automatically delete head branches" setting) rather than left to individual judgment — the same mechanical-enforcement argument (Principle 8) already made for other invariants in this handbook. Where the platform toggle genuinely can't be used — legacy self-hosted setups, unusual multi-repo configurations — a scheduled bot pruning branches inactive past a fixed window (commonly around two weeks) is the fallback, not an excuse to skip enforcement.

**When to choose each:** Automatic deletion is the default for essentially every production repository. Manual deletion is only defensible where no automated option exists yet, and should be treated as a gap to close, not a stable policy. Retention is rare and usually a symptom of using the wrong tool — regulated environments that genuinely need a permanent record of a point-in-time state are almost always better served by a tag or release artifact (Ch 56) than by keeping a branch pointer alive indefinitely.

**Common failure modes:** The graveyard collapse — a repository leaves merged branches in place under the assumption that retaining them is harmless, and over a couple of years accumulates well over a thousand stale remote branches. During a production incident, an on-call engineer runs a branch listing looking for an in-flight hotfix and gets a wall of unreadable, undifferentiated noise, adding real time to an already time-critical recovery. A quieter variant: developers start reusing old, half-forgotten branches instead of creating new ones, reintroducing hidden state nobody remembers exists.

**Example:** GitHub's and GitLab's "automatically delete head branches" setting is not a cosmetic convenience — it's the structural enforcement that keeps a repository's branch namespace equal to its active work, nothing more. Deleting a branch ref this way doesn't destroy any commit that was actually merged; it removes a pointer that had already served its purpose the moment the merge completed.

---

### Why Smart Engineers Disagree

The disagreement isn't about naming — it's about whether a branch is disposable execution state or a form of lightweight, persistent record worth keeping around.

One view treats a Git branch as nothing more than a lightweight pointer to a commit — deleting it removes the pointer, not the commit, provided that commit is already reachable from somewhere durable (the target branch's history, or a tag). From this view, a merged branch has delivered its entire value the moment it merges, and keeping the pointer alive afterward serves no purpose except adding one more entry to a list someone eventually has to search through. The opposing view treats a branch as a form of cheap insurance: if a feature needs to be rolled back, or a discarded approach explored during review turns out to be worth revisiting, having the branch still there feels like a free safety net.

The resolution turns on what's actually true about Git's mechanics, not intuition about "deleting" something. A branch ref pointing at commits that already merged into the target branch protects nothing — those commits are already preserved wherever the merge landed them, with or without the now-redundant pointer. The safety-net argument only has real force for work that never merged and exists nowhere else — and even there, a stale, months-old branch has usually drifted so far from the current codebase that reviving it costs more than rewriting the change from scratch. Where genuine long-term retention is the actual goal — an audit requirement, a release snapshot — a tag or release artifact (Ch 56) is the tool built for that job; a branch left alive indefinitely is a workaround standing in for a mechanism the team hasn't adopted yet.
